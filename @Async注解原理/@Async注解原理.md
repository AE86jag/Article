```java

ConfigurationClassParser.processImports(ConfigurationClass configClass, SourceClass currentSourceClass,
			Collection<SourceClass> importCandidates, Predicate<String> exclusionFilter,
			boolean checkForCircularImports)
```





实践：

异步方法使用当前登录信息，导致Token丢失；重写Executor；使用InheritThreadLocal