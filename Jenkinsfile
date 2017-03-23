
/**
 * [build description]
 * @param  arch 'arm', 'x86', 'arm64', or 'x64'
 * @param  mode [description]
 * @return      [description]
 */
def build(arch, mode) {
  return {
    node('(osx || linux) && git && android-ndk && android-sdk && python && ninja') {
      unstash 'sources'

      dir('v8') {
        // On Mac, we need to hack the NDK/SDK used, since it uses linux version. So create symbolic link to pre-installed versions we have
        // FIXME On linux, we could just add target_os = ['android'] back into .gclient and run gclient sync?
        def os = sh(returnStdout: true, script: 'uname').trim()
        // if ('Darwin'.equals(os)) {
          sh 'mkdir third_party/android_tools'
          sh 'ln -s /opt/android-ndk-r12b third_party/android_tools/ndk'
          sh 'ln -s /opt/android-sdk third_party/android_tools/sdk'
        // } else {
          // TODO hack .gclient to add back android os as target and do gclient sync?
        // }

        def builderName = 'V8 Android Arm - builder'
        if ('x86' == arch) {
          builderName = 'V8 Android x86 - builder'
        } else if ('x64' == arch) {
          builderName = 'V8 Android x64 - builder'
        } else if ('arm64' == arch) {
          builderName = 'V8 Android Arm64 - builder'
        }

        // Generate the build scripts for the target
        sh "tools/dev/v8gen.py gen --no-goma -b '${builderName}' -m client.v8.ports android_${arch}.${mode} -- v8_enable_i18n_support=false symbol_level=0"

        // Build!
        sh "ninja -C out.gn/android_${arch}.${mode} -j 8 v8_nosnapshot v8_libplatform"

        // See v8/build/config/android/config.gni
        def hostOS = 'linux-x86_64'
        if ('Darwin'.equals(os)) {
          hostOS = 'darwin-x86_64'
        }
        def parentDir = 'arm-linux-androideabi'
        def toolchain = 'arm-linux-androideabi-4.9'
        if ('x86' == arch) {
          toolchain = 'x86-4.9'
          parentDir = 'i686-linux-android'
        } else if ('x64' == arch) {
          toolchain = 'x86_64-4.9'
          parentDir = 'x86_64-linux-android'
        } else if ('arm64' == arch) {
          toolchain = 'aarch64-linux-android-4.9'
          parentDir = 'aarch64-linux-android'
        }

        def arPath = "third_party/android_tools/ndk/toolchains/${toolchain}/prebuilt/${hostOS}/${parentDir}/bin/ar"
        // Now make the 'fat' libraries
        sh "${arPath} -rcsD libv8_base.a out.gn/android_${arch}.${mode}/obj/v8_base/*.o"
        sh "${arPath} -rcsD libv8_libbase.a out.gn/android_${arch}.${mode}/obj/v8_libbase/*.o"
        sh "${arPath} -rcsD libv8_libsampler.a out.gn/android_${arch}.${mode}/obj/v8_libsampler/*.o"
        sh "${arPath} -rcsD libv8_libplatform.a out.gn/android_${arch}.${mode}/obj/v8_libplatform/*.o"
        sh "${arPath} -rcsD libv8_nosnapshot.a out.gn/android_${arch}.${mode}/obj/v8_nosnapshot/*.o"
      }
      // Copy 'fat' libs to final dir structure
      sh "mkdir -p build/${mode}/libs/${arch}"
      sh "cp v8/libv8_*.a build/${mode}/libs/${arch}"
      stash includes: "build/${mode}/**", name: "results-${arch}-${mode}"
    }
  }
}

timestamps {
  def gitRevision = '' // we calculate this later for the v8 repo
  // FIXME How do we get the current branch in a detached state?
  def gitBranch = '5.7-lkgr'
  def timestamp = '' // we generate this later
  def v8Version = '' // we calculate this later from the v8 repo
  def modes = ['release', 'debug']
  def arches = ['arm', 'x86']

  node('(osx || linux) && git && python') {
    stage('Checkout') {
      // checkout scm
      // Hack for JENKINS-37658 - see https://support.cloudbees.com/hc/en-us/articles/226122247-How-to-Customize-Checkout-for-Pipeline-Multibranch
      checkout([
        $class: 'GitSCM',
        branches: scm.branches,
        extensions: scm.extensions + [
          [$class: 'CleanBeforeCheckout'],
          [$class: 'SubmoduleOption', disableSubmodules: false, parentCredentials: true, recursiveSubmodules: true, reference: '', timeout: 60, trackingSubmodules: false],
          [$class: 'CloneOption', depth: 30, honorRefspec: true, noTags: true, reference: '', shallow: true]
        ],
        userRemoteConfigs: scm.userRemoteConfigs
      ])

      if (!fileExists('depot_tools')) {
        sh 'mkdir depot_tools'
        dir('depot_tools') {
          git 'https://chromium.googlesource.com/chromium/tools/depot_tools.git'
        }
      }
    } // stage

    stage('Setup') {
      // Grab some values we need for the libv8.json file when we package at the end
      dir('v8') {
        gitRevision = sh(returnStdout: true, script: 'git rev-parse HEAD').trim()
        timestamp = sh(returnStdout: true, script: 'date \'+%Y-%m-%d %H:%M:%S\'').trim()
        // build the v8 version
        def MAJOR = sh(returnStdout: true, script: 'grep "#define V8_MAJOR_VERSION" "include/v8-version.h" | awk \'{print $NF}\'').trim()
        def MINOR = sh(returnStdout: true, script: 'grep "#define V8_MINOR_VERSION" "include/v8-version.h" | awk \'{print $NF}\'').trim()
        def BUILD = sh(returnStdout: true, script: 'grep "#define V8_BUILD_NUMBER" "include/v8-version.h" | awk \'{print $NF}\'').trim()
        def PATCH = sh(returnStdout: true, script: 'grep "#define V8_PATCH_LEVEL" "include/v8-version.h" | awk \'{print $NF}\'').trim()
        v8Version = "${MAJOR}.${MINOR}.${BUILD}.${PATCH}"
      }

      // Add build configs for non-ARM Android, and don't grab Android SDK/NDK
      sh 'git apply android-x86.patch'

      withEnv(["PATH+DEPOT_TOOLS=${env.WORKSPACE}/depot_tools"]) {
        dir('v8') {
          sh 'gclient sync --shallow --no-history --reset --force' // needs python
        } // dir
      } // withEnv

      // stash everything but depot_tools in 'sources'
      stash excludes: 'depot_tools/**', name: 'sources'
      stash includes: 'v8/include/**', name: 'include'
    } // stage
  } // node

  stage('Build') {
    def branches = [failFast: true]
    for (int m = 0; m < modes.size(); m++) {
      def mode = modes[m];
      for (int a = 0; a < arches.size(); a++) {
        def arch = arches[a];
        branches["${arch} ${mode}"] = build(arch, mode);
      }
    }
    parallel(branches)
  } // stage

  node('osx || linux') {
    stage('Package') {
      // unstash v8/include/**
      unstash 'include'

      // Unstash the build artifacts for each arch/mode combination
      for (int m = 0; m < modes.size(); m++) {
        def mode = modes[m];
        for (int a = 0; a < arches.size(); a++) {
          def arch = arches[a];
          unstash "results-${arch}-${mode}"
        }
      }

      // Package each mode
      for (int m = 0; m < modes.size(); m++) {
        def mode = modes[m];

        // write out a JSON file with some metadata about the build
        writeFile file: "build/${mode}/libv8.json", text: """{
	"version": "${v8Version}",
	"git_revision": "${gitRevision}",
	"git_branch": "${gitBranch}",
	"svn_revision": "",
	"timestamp": "${timestamp}"
}
"""
        sh "mkdir -p 'build/${mode}/libs' 'build/${mode}/include' 2>/dev/null"
        sh "cp -R 'v8/include' 'build/${mode}'"
        dir("build/${mode}") {
          echo "Building libv8-${v8Version}-${mode}.tar.bz2..."
          sh "tar -cvj -f libv8-${v8Version}-${mode}.tar.bz2 libv8.json libs include"
          archiveArtifacts "libv8-${v8Version}-${mode}.tar.bz2"
        }
      }
    } // stage

    stage('Publish') {
      if (!env.BRANCH_NAME.startsWith('PR-')) {
        // Publish each mode to S3
        for (int m = 0; m < modes.size(); m++) {
          def mode = modes[m];
          def filename = "build/${mode}/libv8-${v8Version}-${mode}.tar.bz2"
          step([
            $class: 'S3BucketPublisher',
            consoleLogLevel: 'INFO',
            entries: [[
              bucket: 'timobile.appcelerator.com/libv8',
              gzipFiles: false,
              selectedRegion: 'us-east-1',
              sourceFile: filename,
              uploadFromSlave: true,
              userMetadata: []
            ]],
            profileName: 'Jenkins',
            pluginFailureResultConstraint: 'FAILURE',
            userMetadata: []])
        }
      }
    } // stage
  } // node
} // timestamps