---
title: JDK8's Nashorn performance issues
tags: [jdk8, nashorn, typescript]
---

# {{ title }}

I'm generally excited about being able to run a language on multiple platforms, it can often save
you writing a lot of duplicate code and simplify your build process.

I was therefore happy when Oracle announced the release of JDK8. While most people talked
about lambdas, I was more interested in the new JavaScript engine Nashorn, which
came [promising to significantly outperform
the old Rhino engine and provide a nicer interop with Java
code](http://java.dzone.com/articles/project-nashorn-javascripts)
the supposedly high-performance JavaScript engine with a nice interface to Java code, thus replacing
Rhino, which, frankly, is unnecessarily painful to work with.

Thus, when [TypeScript 1.0](http://blogs.msdn.com/b/typescript/archive/2014/04/02/announcing-typescript-1-0.aspx)
came out I thought it'd be great to make TypeScript compile fast on the JVM at last,
so I jumped straight in and [hacked the compiler to support Java's
IO via Nashorn](https://github.com/coudy/typescript/compare/9ac01de...1.0-nashorn).

However, when I tried compiling two simple lines of TypeScript with `jake && jjs built/local/tsc.js -- hello.ts`
it actually took 90 seconds for my `hello.js` to appear, compared with three seconds using Node.js. Something wasn't
right. I suspected an integration cock-up on my part but no amount of head scratching pointed in
that direction.

## Trying Octane

Then I remembered that Google's [Octane Benchmark](http://octane-benchmark.googlecode.com/svn/latest/index.html)
also uses TypeScript as one of the tests, and tried running it as follows:

    svn checkout http://octane-benchmark.googlecode.com/svn/trunk/ octane-benchmark
    cd octane-benchmark
    jjs run.js

The results were similarly poor.

To rule out any possible `jjs`-specific issues I tried running Octane inside a normal Java application:

    package eu.coudy.nashorn;

    import jdk.nashorn.api.scripting.NashornScriptEngine;

    import javax.script.CompiledScript;
    import javax.script.ScriptEngineManager;
    import javax.script.ScriptException;
    import java.io.IOException;
    import java.nio.file.Files;
    import java.nio.file.Paths;

    // svn checkout http://octane-benchmark.googlecode.com/svn/trunk/ octane-benchmark
    // ...and run in the octane-benchmark directory
    public class OctaneBenchmarkNashorn {
        public static void main(String[] args) {
            NashornScriptEngine nashorn = (NashornScriptEngine) new ScriptEngineManager()
                    .getEngineByName("nashorn");
            try {
                CompiledScript scr = nashorn.compile(new String(Files.readAllBytes(
                        Paths.get("run.js"))));
                scr.eval();
            } catch (ScriptException | IOException e) {
                e.printStackTrace();
            }
        }
    }

Same story.

At this stage I decided to find out whether other people have had the same experience and the results
were mixed: [One pure Nashorn benchmark](https://gist.github.com/hakobera/9802734) (look for Avatar.js results) confirmed my
experience, while [these
tests](http://wnameless.wordpress.com/2013/12/10/javascript-engine-benchmarks-nashorn-vs-v8-vs-spidermonkey/)
on a slightly older build of JDK 8 using a JavaFX WebView yielded a much better score.

## Profiling

Firing up YourKit, I profiled my original TypeScript compilation test and found that most of the
time was spent in rewiring `CallSite`s and other `invokedynamic` related logic:

![91% of time spent in LambdaForm...linkToCallSite]({{urls.media}}/yourkit-typescript-nashorn.png)

This is something Nashorn probably wouldn't have to do if it spotted that most references don't change.
However, that is made harder by the fact that the TypeScript compiler code uses TypeScript's internal module
system, which exposes the module objects to the whole enviroment, so the JS engine must suspect
that those module objects have changed or monkey-patched at any time.
So it might be hard to spot that unless you're say V8, which has been around for years. But of course,
that is just a speculation, and most JavaScript is written like that, so Nashorn should definitely
deal with that.

Curious to find out why Nashorn performs much better in a JavaFX app, I profiled the following code:

    package eu.coudy.fx8;

    import javafx.application.Application;
    import javafx.geometry.HPos;
    import javafx.geometry.VPos;
    import javafx.scene.Scene;
    import javafx.scene.layout.Region;
    import javafx.scene.web.WebEngine;
    import javafx.scene.web.WebView;
    import javafx.stage.Stage;

    public class OctaneBenchmarkJavaFX8 extends Application {

        @Override
        public void start(Stage stage) {
            Scene scene = new Scene(new Browser(), 900, 900);
            stage.setScene(scene);
            stage.show();
        }

        public static void main(String[] args) {
            launch(args);
        }
    }

    class Browser extends Region {

        final WebView browser = new WebView();
        final WebEngine webEngine = browser.getEngine();

        public Browser() {
            webEngine.load("http://octane-benchmark.googlecode.com/svn/latest/index.html");
            //webEngine.load("http://jsconsole.com/");
            getChildren().add(browser);
        }

        @Override
        protected void layoutChildren() {
            layoutInArea(browser, 0, 0, getWidth(), getHeight(), 0, HPos.CENTER, VPos.CENTER);
        }
    }

There were no stack traces available for the rendering or JavaScript evaluation:

![invokeLaterDispatcher taking nearly all of the time]({{urls.media}}/yourkit-octane-javafx.png)

## Ha! It's not Nashorn at all!

Suspecting it wasn't actually Nashorn but something native, I checked a few things using JSConsole.
The WebView clearly behaved differently from `jjs`:

- It didn't have Mozilla's syntax extensions
- Setting the `__proto__` property didn't barf
- `java` & `Java` objects didn't exist
- Java interop code used types from the bizarrely named `netscape.javascript` package

Checking out the JDK8 code (`hg clone http://hg.openjdk.java.net/openjfx/8/master/rt`) I found out
`WebView` was basically WebKit with its stock JavaScriptCore engine and wrappers for Java interop.

This feels completely unlike [the early PR](http://java.dzone.com/articles/project-nashorn-javascripts)
and while Oracle are now apparently improving Nashorn by caching the JIT'ed
bytecode to optimise server-side applications, the cache doesn't persist across processes, so this
won't solve the general sluggishness.

This is a real shame and it means we're not quite there yet when it comes to an easily integratable, debuggable
high performance JavaScript engine for the JVM, but at least there's JavaFX,
which could probably be hacked with a fake `WebView` to work as a pure JavaScript engine
(I'm certainly going to try that), and you can obviously still use Rhino, which [will in many cases be
faster](https://bugs.openjdk.java.net/browse/JDK-8019254?focusedCommentId=13360855&page=com.atlassian.jira.plugin.system.issuetabpanels:comment-tabpanel#comment-13360855).

I'd still argue this isn't bad for a first cut, so well done, Oracle, and hope you fix performance
issues soon, because that'd be awesome!
