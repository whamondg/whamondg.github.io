

Create a new dir and cd into it


gradle init --type groovy-library

edit the build.gradle file to add the 'application' plugin

iapply plugin: 'application'
  2 apply plugin: 'groovy'
  3
  4 repositories {
  5     jcenter()
  6 }
  7
  8 dependencies {
  9     compile 'org.codehaus.groovy:groovy-all:2.4.7'
 10
 11     testCompile 'org.spockframework:spock-core:1.0-groovy-2.4'
 12     testCompile 'junit:junit:4.12'
 13 }











