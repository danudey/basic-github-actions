# java

```sh
cp universal/workflows/*.yml .github/workflows/
cp java/workflows/*.yml      .github/workflows/
cp java/dependabot.yml       .github/dependabot.yml
cp universal/release-please-config.json universal/.release-please-manifest.json .
```

Then set `release-type: maven` in `release-please-config.json`, and enable
`language: java-kotlin, build-mode: autobuild` in `codeql.yml`.

Also: delete whichever of the `maven` and `gradle` entries in `dependabot.yml`
does not apply.

## Maven or Gradle

Every workflow here has one knob:

```yaml
env:
  BUILD_TOOL: "maven"   # maven | gradle
```

The pattern is a `case` over the tool that writes a `$BUILD` command into
`$GITHUB_ENV`; later steps use `$BUILD verify` rather than hard-coding anything.
The wrapper (`./mvnw`, `./gradlew`) is preferred when committed and the runner's
own binary used otherwise.

**Commit the wrapper.** Without one the build uses whatever version the runner
ships, which changes under you.

For Gradle, `gradle/actions/setup-gradle` is used with `validate-wrappers: true`.
That is not optional hygiene: `gradle-wrapper.jar` is executable code committed
to the repository, and a tampered one runs arbitrary code with the build's full
privileges. It has been a real attack.

## The values to change

| workflow | variable | what it is |
|---|---|---|
| all | `BUILD_TOOL` | `maven` or `gradle` |
| all | `WORKING_DIRECTORY` | the directory holding `pom.xml` or `build.gradle` |
| all | `DISTRIBUTION` | `temurin`, or `graalvm` for a native image |
| `ci.yml` | `matrix.java` | every version you support |
| `audit.yml` | `SEVERITY` | the floor that fails the build |

### The JDK matrix

`ci.yml`'s `matrix.java` should agree with `maven.compiler.release` or Gradle's
`sourceCompatibility`. Nothing checks that, and a version claimed but not tested
is the usual source of "it does not run on 17".

The default is the three current LTS releases (17, 21, 25), full matrix on Linux
and the oldest and newest elsewhere — because what the other platforms test is
the filesystem (path separators, case sensitivity, file locking on Windows)
rather than the JDK.

### Architectures

The JVM abstracts the CPU away, so for pure Java the OS matrix is what matters.
Where architecture genuinely bites is JNI or a native dependency, and the
`ubuntu-24.04-arm` row is what covers that.

## What the workflows need from the repository

### SpotBugs with SARIF output

`codequality.yml`'s SpotBugs job will run without this but produce no SARIF, and
you will get a warning saying so. For Maven:

```xml
<plugin>
  <groupId>com.github.spotbugs</groupId>
  <artifactId>spotbugs-maven-plugin</artifactId>
  <version>4.9.6.0</version>
  <configuration>
    <!-- This is the line the workflow needs. -->
    <sarifOutput>true</sarifOutput>
    <includeTests>true</includeTests>
    <effort>Max</effort>
    <threshold>Medium</threshold>
    <plugins>
      <!-- Injection, weak crypto, path traversal, XXE, deserialisation.
           This is the security half; without it SpotBugs is a correctness
           tool only. -->
      <plugin>
        <groupId>com.h3xstream.findsecbugs</groupId>
        <artifactId>findsecbugs-plugin</artifactId>
        <version>1.14.0</version>
      </plugin>
    </plugins>
  </configuration>
</plugin>
```

For Gradle:

```groovy
plugins { id 'com.github.spotbugs' version '6.4.4' }

spotbugs {
  effort = com.github.spotbugs.snom.Effort.MAX
  reportLevel = com.github.spotbugs.snom.Confidence.MEDIUM
}
dependencies {
  spotbugsPlugins 'com.h3xstream.findsecbugs:findsecbugs-plugin:1.14.0'
}
tasks.withType(com.github.spotbugs.snom.SpotBugsTask).configureEach {
  reports { sarif.required = true }
}
```

**Why SpotBugs as well as CodeQL.** CodeQL analyses source; SpotBugs analyses
bytecode, and finds a different class of defect — a null dereference on a path
the source does not make obvious, `equals` without `hashCode`, a stream never
closed, two `Integer`s compared with `==`. Enable both.

Note that a multi-module project produces one SARIF per module. The workflow
collects them into a directory and flattens the module path into each filename,
so two modules' reports do not overwrite each other.

### Spotless

`fmt.yml` runs `spotless:check` / `spotlessCheck`. Without the plugin it fails
with "no plugin found", which is a confusing way to learn formatting is not
configured.

```xml
<plugin>
  <groupId>com.diffplug.spotless</groupId>
  <artifactId>spotless-maven-plugin</artifactId>
  <version>3.3.0</version>
  <configuration>
    <java>
      <googleJavaFormat><style>GOOGLE</style></googleJavaFormat>
      <removeUnusedImports/>
      <importOrder><order>java,javax,org,com,</order></importOrder>
      <formatAnnotations/>
    </java>
    <pom>
      <sortPom>
        <expandEmptyElements>false</expandEmptyElements>
      </sortPom>
    </pom>
    <markdown><flexmark/></markdown>
  </configuration>
</plugin>
```

```groovy
plugins { id 'com.diffplug.spotless' version '8.0.0' }

spotless {
  java {
    googleJavaFormat()
    removeUnusedImports()
    importOrder 'java', 'javax', 'org', 'com', ''
  }
  // The build scripts themselves; Gradle has no other convention for this.
  groovyGradle {
    target '*.gradle', 'gradle/**/*.gradle'
    greclipse()
  }
  format 'misc', { target '*.md', '.gitignore'; trimTrailingWhitespace(); endWithNewline() }
}
```

### A `release` profile, for Maven

`release.yml` runs `-Prelease deploy`. Put the GPG signing and javadoc plugin
executions in that profile so an ordinary local build does not need the signing
key:

```xml
<profiles>
  <profile>
    <id>release</id>
    <build><plugins>
      <plugin>
        <groupId>org.apache.maven.plugins</groupId>
        <artifactId>maven-gpg-plugin</artifactId>
        <executions><execution>
          <id>sign-artifacts</id><phase>verify</phase>
          <goals><goal>sign</goal></goals>
        </execution></executions>
      </plugin>
    </plugins></build>
  </profile>
</profiles>
```

## Publishing: this is the one that needs secrets

Every other release workflow in this library publishes without a stored
credential, because PyPI, npm, crates.io and GHCR all accept a workflow OIDC
token. **Maven Central does not.** It needs four secrets, and you should store
them on the `maven-central` GitHub environment rather than on the repository, so
only that job can read them:

| secret | what it is |
|---|---|
| `MAVEN_CENTRAL_USERNAME` | a Central Portal **user token**, not your password |
| `MAVEN_CENTRAL_PASSWORD` | the other half of that token pair |
| `GPG_PRIVATE_KEY` | ASCII-armoured private key; Central rejects unsigned uploads |
| `GPG_PASSPHRASE` | for the above |

Generate the token pair at <https://central.sonatype.com/account>. Generate the
GPG key with:

```sh
gpg --full-generate-key                      # RSA 4096, no expiry or a long one
gpg --armor --export-secret-keys KEYID       # → GPG_PRIVATE_KEY
gpg --keyserver keyserver.ubuntu.com --send-keys KEYID   # required by Central
```

The GPG key is the one real long-lived secret in this library. Keep the private
half nowhere else, add a required reviewer to the `maven-central` environment,
and rotate it if a run is ever compromised.

The `server-id: central` on `setup-java` has to match the `<id>` in the pom's
`<distributionManagement>`, or Maven looks for credentials it cannot find and
fails with a 401.

### GitHub Packages instead

Much simpler and needs no secrets at all: `secrets.GITHUB_TOKEN` with
`packages: write`, and no GPG signature required. There is a commented
`publish-github-packages` job at the bottom of `release.yml`.

The trade-off: consumers must add your GitHub Packages repository to their
`settings.xml` **and authenticate to it**, even for a public package. For an
internal library that is fine; for a public one it is usually a worse deal than
Central.

### Reproducible builds

`ci.yml` has a `reproducible` job, disabled with `if: false`. Turn it on once
you have configured for it, because it matters more than it sounds:

A jar embeds timestamps and file order by default, so two builds of the same
commit produce different bytes. That makes the release attestations weaker than
they look — a verifier can check *who* built the jar but not that the source
matches. The fix is usually two properties:

```xml
<properties>
  <project.build.outputTimestamp>2026-01-01T00:00:00Z</project.build.outputTimestamp>
</properties>
```

`mvn artifact:check-buildplan` reports whether the project is configured
correctly.

## Dependency auditing

`audit.yml` uses osv-scanner and Trivy rather than OWSAP dependency-check.
That is deliberate: dependency-check needs an NVD API key, downloads a
multi-gigabyte database, and takes ten minutes on a cold cache. There is a
commented job for it if a compliance process requires its report specifically.

Java's real problem is transitive depth. A Spring application resolves several
hundred jars from a dozen direct dependencies, and the vulnerable one is almost
always something nobody chose. Trivy is the better of the two here because it
reports **which direct dependency pulled the vulnerable one in**, which is the
piece of information you actually need to fix it.

**For a Gradle project, enable dependency locking.** Without a lockfile or
`gradle/verification-metadata.xml`, osv-scanner sees the declared ranges rather
than what resolves, and the scan is much weaker:

```groovy
dependencyLocking { lockAllConfigurations() }
```

then `./gradlew dependencies --write-locks` and commit `gradle.lockfile`.

## Things that will bite you

**`spotbugs:check` finds nothing and produces no SARIF.** `sarifOutput` is not
enabled. See above; the workflow warns about this explicitly.

**A Dependabot major bump breaks everything.** In Java a major version is
frequently a package rename — `javax` to `jakarta` being the famous one. That is
why `dependabot.yml` splits runtime dependencies by update type: a major belongs
in its own reviewed pull request rather than blocking a batch of patches.

**Dependabot does not update the Gradle distribution.** `gradle-wrapper.properties`
is not in scope. Run `./gradlew wrapper --gradle-version=X` and commit the
result.

**A Dependabot bump fails only on the Java 17 row.** A dependency now requires a
newer JDK than you target. Either raise the target (a decision about who you
support) or hold the dependency back.

**The test failure log says nothing useful.** A test that fails inside a forked
JVM prints nothing to the console; the surefire XML is the only record. `ci.yml`
uploads those reports with `if: always()` for exactly this.

**`mvn deploy` fails with a 401.** Either `server-id` does not match the pom, or
you are using your account password instead of a Central Portal user token.
