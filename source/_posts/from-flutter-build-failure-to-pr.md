---
title: 从Flutter编译错误到给Flutter贡献代码
date: 2021-03-21 13:55:38
tags: [Flutter, i18n]
typora-root-url: ./from-flutter-build-failure-to-pr/
---

# Flutter国际化

[Flutter的官方文档](https://flutter.dev/docs/development/accessibility-and-localization/internationalization)中给出了Flutter应用的国际化和本地化流程。

简而言之，Flutter的国际化依赖代码生成。以JSON格式在指定文件中定义国际化的资源，然后通过Flutter提供的工具动态生成Flutter代码，随后便可以在应用中使用这些代码中的变量。

在根目录（注意[不是lib目录下](https://github.com/flutter/flutter/issues/75977)www）下的l10n.yaml中配置本地化配置：

```yaml
arb-dir: lib/l10n
template-arb-file: app_en.arb
output-localization-file: app_localizations.dart
```

然后在lib/l10n中添加各语言的本地化资源文件，其中作为模板的app_en.arb如下所示：

```json
{
  "appName": "Fandori",
  "@appName": {},
  "information": "Information",
  "@information": {},
  "characters": "Characters",
  "@characters": {},
  "cards": "Cards",
  "@cards": {}
}
```

而其它语言文件就不需要`@`开头的描述性字段了：

```json
{
  "appName": "Fandori",
  "information": "信息",
  "characters": "角色",
  "cards": "卡牌"
}
```

在pubspec.yaml中启用本地化：

```yaml
dependencies:
  flutter_localizations:
    sdk: flutter
  intl: ^0.16.1
flutter:
  generate: true
```

这之后每次运行`flutter pub get`时，都会触发本地化资源的生成。

# JSON序列化

众所周知，[Flutter禁止了反射](https://flutter.dev/docs/development/data-and-backend/json#is-there-a-gsonjacksonmoshi-equivalent-in-flutter)。因此，JSON序列化也需要使用代码生成来实现。

在pubspec.yaml中添加JSON序列化相关的配置：

```yaml
dependencies:
  json_annotation: ^3.1.1
dev_dependencies:
  build_runner: ^1.11.1
  json_serializable: ^3.5.1
```

然后定义需要支持JSON序列化、反序列化的Model类：

```dart
import 'package:json_annotation/json_annotation.dart';

part 'get_band.g.dart';

@JsonSerializable()
class Band {
  int bandId;
  String bandName;
  String introductions;

  Band(
    this.bandId,
    this.bandName,
    this.introductions,
  );

  factory Band.fromJson(Map<String, dynamic> json) => _$BandFromJson(json);

  Map<String, dynamic> toJson() => _$BandToJson(this);
}
```

对于任意文件`some_file.dart`中使用`JsonSerializable`注解标记的类`SomeClass`，`json_serializable`会在`some_file.g.dart`中生成用于序列化的`_$SomeClassToJson`和用于反序列化的`$_SomeClassFromJson`方法。

只需要运行`flutter pub run build_runner build`就可以生成上述`.g.dart`文件。

# build_runner与flutter_generate的冲突

如果你使用的是1.24以前（不含1.24）的Flutter版本，那么在你运行build_runner的时候，便会遇到这样的错误：

```
Bad state: Unable to generate package graph, no /home/perqin/Workspaces/fandori/.dart_tool/flutter_gen/pubspec.yaml found.
```

完整的错误现场可以看GitHub上[另一位老哥发的issue](https://github.com/flutter/flutter/issues/75333)。

这个未被找到的文件位于项目根目录下的`.dart_tool/flutter_gen/`。查看目录下的文件就会发现，这里面就是Flutter本地化工具生成的代码存放处。

于是我产生了两个疑问：

* build_runner为什么会检查这个目录下面有没有pubspec.yaml？
* Flutter为什么没有在这个目录下生成pubspec.yaml？

# 探索build_runner

你可以在Flutter安装目录下的.pub-cache\hosted\pub.dartlang.org中找到build_runner的源码（如果你使用了Flutter China的镜像的话，你要在hosted目录下找pub.flutter-io.cn目录）。

顺着错误栈，我们找到了build_runner入口的第一行代码：

```dart
// build_runner:build_runner.dart
Future<void> main(List<String> args) async {
  // Use the actual command runner to parse the args and immediately print the
  // usage information if there is no command provided or the help command was
  // explicitly invoked.
  var commandRunner =
      BuildCommandRunner([], await PackageGraph.forThisPackage());
  // ...
}
```

在进行后续逻辑之前，build_runner会先获取我们项目的包依赖图（Package Graph）。顺着错误栈找下去吧（中文注释是我加上去的）：

```dart
// build_runner_core:src/package_graph/package_graph.dart
class PackageGraph {
  /// Creates a [PackageGraph] for the package in which you are currently
  /// running.
  static Future<PackageGraph> forThisPackage() =>
      // 读取项目的PackageGraph
      PackageGraph.forPath(p.current);

  /// Creates a [PackageGraph] for the package whose top level directory lives
  /// at [packagePath] (no trailing slash).
  static Future<PackageGraph> forPath(String packagePath) async {
    // ...
    // 读取PackageConfig，Flutter会把项目的包依赖信息放在PackageConfig里
    final packageConfig =
        await findPackageConfig(Directory(packagePath), recurse: false);
    // ...
    // 通过PackageConfig解析包依赖
    final packageDependencies = _parsePackageDependencies(
        packageConfig.packages.where((p) => p.name != rootPackageName));
    // ...
}

/// Read the pubspec for each package in [packages] and finds it's
/// dependencies.
Map<String, List<String>> _parsePackageDependencies(
    Iterable<Package> packages) {
  final dependencies = <String, List<String>>{};
  for (final package in packages) {
    // 针对每个包，读取它的pubspec.yaml
    final pubspec = _pubspecForPath(package.root.toFilePath());
    dependencies[package.name] = _depsFromYaml(pubspec);
  }
  return dependencies;
}

/// Should point to the top level directory for the package.
YamlMap _pubspecForPath(String absolutePath) {
  var pubspecPath = p.join(absolutePath, 'pubspec.yaml');
  var pubspec = File(pubspecPath);
  if (!pubspec.existsSync()) {
    // .dart_tool/flutter_gen这个包也被列入了依赖，而这个包里没有pubspec.yaml，所以在这里报了错
    throw StateError(
        'Unable to generate package graph, no `$pubspecPath` found.');
  }
  return loadYaml(pubspec.readAsStringSync()) as YamlMap;
}
```

那么为什么flutter_gen目录会被当作一个包被列入包依赖图的生成呢？package_config这个包中有答案：

```dart
// package_config:package_config.dart
Future<PackageConfig> findPackageConfig(Directory directory,
        {bool recurse = true, void onError(Object error)}) =>
    discover.findPackageConfig(directory, recurse, onError ?? throwError);
```

```dart
// package_config:src/discovery.dart
Future<PackageConfig /*?*/ > findPackageConfig(
    Directory baseDirectory, bool recursive, void onError(Object error)) async {
  var directory = baseDirectory;
  if (!directory.isAbsolute) directory = directory.absolute;
  if (!await directory.exists()) {
    return null;
  }
  do {
    // Check for $cwd/.packages
    // 读取项目根目录下的.packages信息
    var packageConfig = await findPackagConfigInDirectory(directory, onError);
    if (packageConfig != null) return packageConfig;
    if (!recursive) break;
    // Check in parent directories.
    var parentDirectory = directory.parent;
    if (parentDirectory.path == directory.path) break;
    directory = parentDirectory;
  } while (true);
  return null;
}
```

现在我们打开项目根目录下的.packages：

```
# generated by package:package_config at 2021-02-19 10:26:07.001909
# 前面省略……
flutter_gen:.dart_tool/flutter_gen/
```

于是我们可以推断出事件的全貌：

Flutter的本地化工具会在.dart_tool目录下生成一个本地的Dart包：flutter_gen，并把生成的本地化代码放入其中。Flutter会把项目的依赖包列表都写入到.packages中，供包括build_runner在内的其他工具使用。

也就是说锅并不是build_runner的，而是Flutter的本地化工具的。

# 探索Flutter Localization

前面提到，在运行`flutter pub get`的时候会刷新Flutter本地化代码，这句话的依据在Flutter安装目录下的packages\flutter_tools\lib\src\commands\packages.dart中：

```dart
class PackagesGetCommand extends FlutterCommand {
  Future<void> _runPubGet(String directory, FlutterProject flutterProject) async {
    if (flutterProject.manifest.generateSyntheticPackage) {
      final Environment environment = Environment(
        artifacts: globals.artifacts,
        logger: globals.logger,
        cacheDir: globals.cache.getRoot(),
        engineVersion: globals.flutterVersion.engineRevision,
        fileSystem: globals.fs,
        flutterRootDir: globals.fs.directory(Cache.flutterRoot),
        outputDir: globals.fs.directory(getBuildDirectory()),
        processManager: globals.processManager,
        projectDir: flutterProject.directory,
      );

      await generateLocalizationsSyntheticPackage(
        environment: environment,
        buildSystem: globals.buildSystem,
      );
      // ...
    }
  }
}
```

那么我是怎么找到这个文件的呢www

在pubspec.yaml中，有这样一个配置：

```yaml
flutter:
  uses-material-design: true
```

在Flutter的GitHub仓库中搜索“uses-material-design”，过滤出Dart文件，很容易就能找到文件packages/flutter_tools/lib/src/flutter_manifest.dart，其中就包含了判断本地化是否启用的代码：

```dart
class FlutterManifest {
  /// Whether a synthetic flutter_gen package should be generated.
  ///
  /// This can be provided to the [Pub] interface to inject a new entry
  /// into the package_config.json file which points to `.dart_tool/flutter_gen`.
  ///
  /// This allows generated source code to be imported using a package
  /// alias.
  bool get generateSyntheticPackage => _generateSyntheticPackage ??= _computeGenerateSyntheticPackage();
  bool _generateSyntheticPackage;
  bool _computeGenerateSyntheticPackage() {
    if (!_flutterDescriptor.containsKey('generate')) {
      return false;
    }
    final Object value = _flutterDescriptor['generate'];
    if (value is! bool) {
      return false;
    }
    return value as bool;
  }
}
```

再顺着generateSyntheticPackage就能找到本地化工具的具体实现了。

回到generateLocalizationsSyntheticPackage：

```dart
// packages/flutter_tools/lib/src/dart/generate_synthetic_packages.dart
Future<void> generateLocalizationsSyntheticPackage({
  @required Environment environment,
  @required BuildSystem buildSystem,
}) async {
  // ...
  final BuildResult result = await buildSystem.build(
    const GenerateLocalizationsTarget(),
    environment,
  );

  if (result == null || result.hasException) {
    throwToolExit('Generating synthetic localizations package has failed.');
  }
}
```

显然，生成flutter_gen的逻辑就在GenerateLocalizationsTarget了：

```dart
// packages/flutter_tools/lib/src/build_system/targets/localizations.dart
class GenerateLocalizationsTarget extends Target {
  @override
  Future<void> build(Environment environment) async {
    // ...
    generateLocalizations(
      logger: environment.logger,
      options: options,
      projectDir: environment.projectDir,
      dependenciesDir: environment.buildDir,
      localizationsGenerator: LocalizationsGenerator(environment.fileSystem),
    );
    // ...
  }
}
```

```dart
// packages/flutter_tools/lib/src/localizations/gen_l10n.dart
/// Run the localizations generation script with the configuration [options].
void generateLocalizations({
  @required Directory projectDir,
  @required Directory dependenciesDir,
  @required LocalizationOptions options,
  @required LocalizationsGenerator localizationsGenerator,
  @required Logger logger,
}) {
  // ...
  try {
    localizationsGenerator
      ..initialize(
        inputsAndOutputsListPath: dependenciesDir?.path,
        projectPathString: projectDir.path,
        inputPathString: inputPathString,
        templateArbFileName: templateArbFileName,
        outputFileString: outputFileString,
        outputPathString: options?.outputDirectory?.path,
        classNameString: options.outputClass ?? 'AppLocalizations',
        preferredSupportedLocales: options.preferredSupportedLocales,
        headerString: options.header,
        headerFile: options?.headerFile?.toFilePath(),
        useDeferredLoading: options.deferredLoading ?? false,
        useSyntheticPackage: options.useSyntheticPackage ?? true,
        areResourceAttributesRequired: options.areResourceAttributesRequired ?? false,
        untranslatedMessagesFile: options?.untranslatedMessagesFile?.toFilePath(),
      )
      ..loadResources()
      ..writeOutputFiles(logger, isFromYaml: true);
  } on L10nException catch (e) {
    logger.printError(e.message);
    throw Exception();
  }
}

class LocalizationsGenerator {
  void writeOutputFiles(Logger logger, { bool isFromYaml = false }) {
    // First, generate the string contents of all necessary files.
    _generateCode();

    // A pubspec.yaml file is required when using a synthetic package. If it does not
    // exist, create a blank one.
    if (_useSyntheticPackage) {
      final Directory syntheticPackageDirectory = _fs.directory(defaultSyntheticPackagePath);
      syntheticPackageDirectory.createSync(recursive: true);
      final File flutterGenPubspec = syntheticPackageDirectory.childFile('pubspec.yaml');
      if (!flutterGenPubspec.existsSync()) {
        flutterGenPubspec.writeAsStringSync(emptyPubspecTemplate);
      }
    }

    // ...
}
```

当时我是在GitHub上看这段代码的，很明显在writeOutputFiles中是包含了pubspec.yaml的生成逻辑的，怎么还会找不到呢？

我们切换到v1.22.6这个tag，再看[这个文件](https://github.com/flutter/flutter/blob/9b2d32b605630f28625709ebd9d78ab3016b2bf6/packages/flutter_tools/lib/src/localizations/gen_l10n.dart#L1003)：

```dart
// ...
void writeOutputFiles() {
    // First, generate the string contents of all necessary files.
    _generateCode();

    // Since all validity checks have passed up to this point,
    // write the contents into the directory.
    if (!outputDirectory.existsSync()) {
      outputDirectory.createSync(recursive: true);
    }

    // Ensure that the created directory has read/write permissions.
    final FileStat fileStat = outputDirectory.statSync();
    if (_isNotReadable(fileStat) || _isNotWritable(fileStat)) {
      throw L10nException(
        "The 'output-dir' directory, $outputDirectory, doesn't allow reading and writing.\n"
        'Please ensure that the user has read and write permissions.'
      );
    }

    // Generate the required files for localizations.
    _languageFileMap.forEach((File file, String contents) {
      file.writeAsStringSync(contents);
      if (_inputsAndOutputsListFile != null) {
        _outputFileList.add(file.absolute.path);
      }
    });

    baseOutputFile.writeAsStringSync(_generatedLocalizationsFile);
    if (_inputsAndOutputsListFile != null) {
      _outputFileList.add(baseOutputFile.absolute.path);

      // Generate a JSON file containing the inputs and outputs of the gen_l10n script.
      if (!_inputsAndOutputsListFile.existsSync()) {
        _inputsAndOutputsListFile.createSync(recursive: true);
      }

      _inputsAndOutputsListFile.writeAsStringSync(
        json.encode(<String, Object> {
          'inputs': _inputFileList,
          'outputs': _outputFileList,
        }),
      );
    }
  }
  // ...
```

可见在此刻的最新版本（此时1.23以及更新的版本都还没有进入stable）上还有这个bug，而最新的master代码已经修复了。我们很容易找到[对应的PR](https://github.com/flutter/flutter/pull/68206)，查看这个文件的改动对应的[commit](https://github.com/flutter/flutter/commit/38ebc5588b3d80b5bcce1a0a8114f96813de593f)可以看到我们最早也要在1.24中才能收到这个修复了：

![](flutter-gen-bugfix-commit.png)

如果你不想等待1.24发布正式版，你也可以：

* 手动创建pubspec.yaml文件
* 切换到beta channel

# Flutter Localization中的另一个bug

在阅读代码的过程中，我留意到GenerateLocalizationsTarget中的代码有些不对劲：

```dart
// packages/flutter_tools/lib/src/dart/generate_synthetic_packages.dart
Future<void> generateLocalizationsSyntheticPackage({
  @required Environment environment,
  @required BuildSystem buildSystem,
}) async {
  assert(environment != null);
  assert(buildSystem != null);

  final FileSystem fileSystem = environment.fileSystem;
  final File l10nYamlFile = fileSystem.file(
    fileSystem.path.join(environment.projectDir.path, 'l10n.yaml'));

  // If pubspec.yaml has generate:true and if l10n.yaml exists in the
  // root project directory, check to see if a synthetic package should
  // be generated for gen_l10n.
  if (!l10nYamlFile.existsSync()) {
    return;
  }

  final YamlNode yamlNode = loadYamlNode(l10nYamlFile.readAsStringSync());
  if (yamlNode.value != null && yamlNode is! YamlMap) {
    throwToolExit(
      'Expected ${l10nYamlFile.path} to contain a map, instead was $yamlNode'
    );
  }

  BuildResult result;
  // If an l10n.yaml file exists but is empty, attempt to build synthetic
  // package with default settings.
  if (yamlNode.value == null) {
    result = await buildSystem.build(
      const GenerateLocalizationsTarget(),
      environment,
    );
  } else {
    final YamlMap yamlMap = yamlNode as YamlMap;
    final Object value = yamlMap['synthetic-package'];
    if (value is! bool && value != null) {
      throwToolExit(
        'Expected "synthetic-package" to have a bool value, '
        'instead was "$value"'
      );
    }

    // Generate gen_l10n synthetic package only if synthetic-package: true or
    // synthetic-package is null.
    final bool isSyntheticL10nPackage = value as bool ?? true;
    if (!isSyntheticL10nPackage) {
      return;
    }
  }

  result = await buildSystem.build(
    const GenerateLocalizationsTarget(),
    environment,
  );

  if (result == null || result.hasException) {
    throwToolExit('Generating synthetic localizations package has failed.');
  }
}
```

可以看到上述代码片段中的31、53行都调用了buildSystem.build，这不是重复调用了？？

为了确认这个问题，首先我将本地Flutter安装目录下的bin/cache/flutter_tools.snapshot删掉，这个文件是Flutter在运行的时候将flutter_tools这个包预先编译成的二进制文件，从而避免每次运行都需要解释执行Dart源码。因此，在修改本地代码之前，需要删掉相应的snapshot文件。

在generate_synthetic_packages.dart文件中的两个build调用前都加上print日志打印，然后观察这段代码，可以看出当`(yamlNode.value == null || yamlNode is YamlMap) && yamlNode.value == null`，也就是yamlNode.value为null的时候能够触发。

在l10n.yaml中为空字符串或者null的时候，yamlNode.value为null，而进一步确认packages/flutter_tools/lib/src/localizations/localizations_utils.dart：

```dart
LocalizationOptions parseLocalizationsOptions({
  @required File file,
  @required Logger logger,
}) {
  final String contents = file.readAsStringSync();
  if (contents.trim().isEmpty) {
    return const LocalizationOptions();
  }
  final YamlNode yamlNode = loadYamlNode(file.readAsStringSync());
  if (yamlNode is! YamlMap) {
    logger.printError('Expected ${file.path} to contain a map, instead was $yamlNode');
    throw Exception();
  }
  // ...
}
```

可见如果l10n.yaml包含注释或者null等内容，则会抛出异常，从而在第一次build调用中就会中止。

因此我们打开l10n.yaml，清空其中的内容，然后运行`flutter pub get`，你就会看到两次build调用的日志打印了。由于这篇文章是拖延了两个月后才写的，当时的现场我懒得复原了www

事实上，只要GenerateLocalizationsTarget的实现是稳定的，那么不论重复运行多少次，应该都能生成一样的本地化相关代码，这个bug并不算严重，但是~~为了环保~~我还是尝试了修复。

修复办法平平无奇，这里就不贴了。

# 将bug掩藏起来的错误的单元测试

Flutter这样大型的开源项目必然会使用单元测试高强度覆盖，以保证代码质量。上面这个bug涉及的代码也是有单元测试的，那么为什么单元测试没有测出这个问题呢？

上述文件对应的测试代码在packages/flutter_tools/test/general.shard/dart/generate_synthetic_packages_test.dart，可以看到第一个测试用例就是l10n.yaml为空的场景：

```dart
void main() {
  testWithoutContext('calls buildSystem.build with blank l10n.yaml file', () {
    // ...
    expect(
      () => generateLocalizationsSyntheticPackage(
        environment: environment,
        buildSystem: buildSystem,
      ),
      throwsToolExit(message: 'Generating synthetic localizations package has failed.'),
    );
    // [BuildSystem] should have called build with [GenerateLocalizationsTarget].
    verify(buildSystem.build(
      const GenerateLocalizationsTarget(),
      environment,
    )).called(1);
  });
  // ...
}
```

我本地运行这个用例，发现这个用例居然是可以通过的！

百思不得其解之际，我在verify处和build调用处都打了断点，意外发现调用顺序如下：

1. 第一次build调用
2. verify
3. 第二次build调用

这就奇怪了，第二次build调用还没开始的时候，generateLocalizationsSyntheticPackage还没有执行完毕，怎么就走到verify了呢？

我随便点开了`throwsToolExit`的实现，发现`throwsToolExit`实际上返回的是`throwsA`：

```dart
// packages/flutter_tools/test/src/common.dart
/// Matcher for functions that throw [ToolExit].
Matcher throwsToolExit({ int exitCode, Pattern message }) {
  Matcher matcher = isToolExit;
  if (exitCode != null) {
    matcher = allOf(matcher, (ToolExit e) => e.exitCode == exitCode);
  }
  if (message != null) {
    matcher = allOf(matcher, (ToolExit e) => e.message?.contains(message) ?? false);
  }
  return throwsA(matcher);
}
```

然后打开Flutter的官方文档：

> # throwsA function
>
> This can be used to match three kinds of objects:
>
> - A [Function](https://api.flutter.dev/flutter/dart-core/Function-class.html) that throws an exception when called. The function cannot take any arguments. If you want to test that a function expecting arguments throws, wrap it in another zero-argument function that calls the one you want to test.
> - A [Future](https://api.flutter.dev/flutter/dart-async/Future-class.html) that completes with an exception. Note that this creates an asynchronous expectation. The call to `expect()` that includes this will return immediately and execution will continue. Later, when the future completes, the actual expectation will run.
> - A [Function](https://api.flutter.dev/flutter/dart-core/Function-class.html) that returns a [Future](https://api.flutter.dev/flutter/dart-async/Future-class.html) that completes with an exception.
>
> In all three cases, when an exception is thrown, this will test that the exception object matches `matcher`. If `matcher` is not an instance of [Matcher](https://api.flutter.dev/flutter/package-matcher_matcher/Matcher-class.html), it will implicitly be treated as `equals(matcher)`.

可以看到，如果被expect的对象是一个Future对象，那么这个expect会立刻返回，而不会等待Future执行完毕。在我们这个场景下，expect的对象不是Future，而是一个返回Future的Function，但阅读throwsA的实现会发现这种情况下expect也是会立刻返回的。

在一般情况下，使用throwsA并没有什么问题，throwsA可以保证测试用例所在的方法执行完毕后仍然保持测试用例处于未结束的状态，等待这个expect获得结果后再结束。

在在这个用例中，我们expect之后马上就调用了verify检查build被调用的次数，显然此时throwsA的这个特性就会带来问题：此时build调用次数确实为1，但后一次调用还在来的路上呢。

查阅expect的文档：

> Certain matchers, like [completion](https://pub.dev/documentation/test_api/latest/test_api/completion.html) and [throwsA](https://pub.dev/documentation/test_api/latest/test_api/throwsA.html), either match or fail asynchronously. When you use [expect](https://pub.dev/documentation/test_api/latest/test_api/expect.html) with these matchers, it ensures that the test doesn't complete until the matcher has either matched or failed. If you want to wait for the matcher to complete before continuing the test, you can call [expectLater](https://pub.dev/documentation/test_api/latest/test_api/expectLater.html) instead and `await` the result.

于是可见，我们应该把测试用例改成：

```dart
void main() {
  // 增加async关键字，我们需要使用await
  testWithoutContext('calls buildSystem.build with blank l10n.yaml file', () async {
    // ...
    // 改用expectLater，并await以保证被expect的方法已经结束并返回
    await expectLater(
      () => generateLocalizationsSyntheticPackage(
        environment: environment,
        buildSystem: buildSystem,
      ),
      throwsToolExit(message: 'Generating synthetic localizations package has failed.'),
    );
    // [BuildSystem] should have called build with [GenerateLocalizationsTarget].
    verify(buildSystem.build(
      const GenerateLocalizationsTarget(),
      environment,
    )).called(1);
  });
  // ...
}
```

这样修改之后，我们的测试用例就失败了，因此也就可以修改代码修复这个bug了。

# 吐槽

throwsA会导致expect异步执行，在和verify搭配使用的时候这样的错误隐蔽又致命，但似乎Flutter和Dart的文档中都没有特别提到这一点，不知道Flutter代码中还有多少这样错误的测试用例代码呢www

另外，明明都是自动生成代码，JSON Annotation库使用的是Build Runner，但Flutter Localization使用的却是Flutter Generate，而且从代码中可以看到这块逻辑几乎完全耦合在Flutter代码中（以至于pubspec.yaml中的`generate`似乎仅仅用于本地化这个模块），为什么不把Localization也交给Build Runner来做呢？