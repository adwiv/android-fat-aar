# Current Status
I am no longer involved in development, so am not able to maintain or provide bug fixes or even test the pull requests. It would be great if someone who actually is using this project in their library and can actively maintain the repo, become part of/take it over.

Meanwhile, you can also check [fat-aar-plugin](https://github.com/Vigi0303/fat-aar-plugin) which is trying to solve the same problem using a plugin.

# android-fat-aar
Gradle script that allows you to merge and embed dependencies in generated aar file. 

In this Fork you can find a fixed version who will also work with embedded .aar files !

**Credits**

[jksiezni](https://github.com/jksiezni) suggested an alternate way to embed R files which fixes ugly internal proguard hack.

[jonbryantnz](https://github.com/jonbryantnz) suggested method to embed java projects.

**Why do I need is a fat AAR?**

There may be multiple reasons for wanting this. My reason was that I wanted to publish a single library 
while maintaining a modular structure within the project. The benefit of a fat aar file is that we can
proguard the combined code instead of proguarding each and every subproject which is not that effective.

**What doesn't work?**

1. Manifest placeholders that are expected to be filled in by the application
2. AIDL File merger - I do not use aidl files
3. Multiple build types - only single build type (release) is supported out of the box
4. ?

**Step 1: Apply the gradle file** 

To use this simply copy the gradle file 'fat-aar.gradle' to your project directory and then

`apply from: 'fat-aar.gradle'`

or apply directly from the url

`apply from: 'https://raw.githubusercontent.com/adwiv/android-fat-aar/master/fat-aar.gradle'`

**Step 2: Define the embedded dependencies** 

Then you can modify the dependencies section and change the word `compile` to `embedded` 
for the dependencies you want merged within the aar file. The resulting section may look like this:

    dependencies {
      compile fileTree(dir: 'libs', include: ['*.jar'])

      // Order of dependencies decide which will have precedence in case of duplicates 
      // during manifest / resource merger 
      embedded project(':librarytwo')
      embedded project(':libraryone')
      embedded project('com.example.internal:lib-three:1.2.3')
      
      compile 'com.example:some-other-lib:1.0.3'
      compile 'com.android.support:appcompat-v7:22.2.0'
    }
    
The dependencies with keyword `embedded` will be merged while the others will remain referenced as before.

**Step 3: Remove embedded dependencies from exported dependency list**

Now that you have embedded your sub projects into the main library, you need to ensure that anyone using 
your library does not resolve the embedded projects as transitive dependencies. Otherwise he will get 
duplicate class errors. 

If you are using the fat library within the same project (maybe within a test app?), then you can simply define 
your fat-library dependency as non transitive.

    compile (project(':applibrary')) {  // Notice the parentheses around project
        transitive false
    }

For external clients or use in another project; this can be achieved by removing these dependencies from the 
generated pom.xml file. How to automate that will depend on how you are generating the pom file. I use 
`maven-publish` plugin with the following pom generator.

    pom.withXml {
        def dependenciesNode = asNode().appendNode('dependencies')
        //Iterate over the compile dependencies (we don't want the test ones), adding a <dependency> node for each
        configurations.compile.allDependencies.each {
            if(it.group != null && (it.name != null || "unspecified".equals(it.name)) && it.version != null)
            {
                if(!configurations.embedded.allDependencies.contains(it)) {
                def dependencyNode = dependenciesNode.appendNode('dependency')
                dependencyNode.appendNode('groupId', it.group)
                dependencyNode.appendNode('artifactId', it.name)
                dependencyNode.appendNode('version', it.version)
                }
            }
        }
    }

**The complete `publish.gradle` file (That also automatically adds the transitive dependencies as primary) is in the repository.**

Hope this helps.
