pipeline {
    agent any
    parameters {
        string(name: 'VERSION', defaultValue: '13.0.0~v2test', description: 'version string to put on stellar-core .deb package')
        string(name: 'DEBFULLNAME', defaultValue: 'Package Maintainer', description:'used by dpkg-buildpackage')
        string(name: 'DEBEMAIL', defaultValue: 'packages@stellar.org', description:'used by dpkg-buildpackage')
        string(name: 'BRANCH', defaultValue: 'origin/master', description:'Use origin/testnet for build tagged as testnet, origin/acceptance-test-pass for latest build that passed acceptance, origin/pr/NUMBER for a build based on a pull request, origin/tags/TAGNAME for a specific tag (like v10.0.0)')
        string(name: 'DEB_CONFIGURE_OPTS', defaultValue: '--prefix=/usr --includedir=/usr/include --mandir=/usr/share/man --infodir=/usr/share/info --sysconfdir=/etc --localstatedir=/var', description:'')
        string(name: 'DEB_CXXFLAGS_SET', defaultValue: '-ggdb -O3 -fstack-protector-strong -Wformat -Werror=format-security', description:'CXXFLAGS is retrieved from `dpkg-buildflags` and is prepended to by the debian/rules file found in `stellar-core` using DEB_CXXFLAGS_MAINT_PREPEND. If you want to override the default CXXFLAGS please modify the DEB_CXXFLAGS_SET parameter. Please not that the DEB_CXXFLAGS_MAINT_PREPEND will still prepend it\'s value to DEB_CXXFLAGS_SET. For reference the debian/rules value of DEB_CXXFLAGS_MAINT_PREPEND is "-stdlib=libc++ -fno-omit-frame-pointer -isystem /usr/include/libcxxabi"')
        string(name: 'DEB_CFLAGS_SET', defaultValue: '-ggdb -O3 -fstack-protector-strong -Wformat -Werror=format-security', description:'CFLAGS is retrieved from `dpkg-buildflags`. If you want to override the default CFLAGS please modify the DEB_CFLAGS_SET parameter.')
        string(name: 'REPO_URL', defaultValue: 'git@github.com:stellar/stellar-core.git', description:'URL to retrieve code from')
    }
    environment {
        BUILD_VERSION = "${params.VERSION}"
        DEBFULLNAME = "${params.DEBFULLNAME}"
        DEBEMAIL = "${params.DEBEMAIL}"
        BRANCH = "${params.BRANCH}"
        DEB_CONFIGURE_OPTS = "${params.DEB_CONFIGURE_OPTS}"
        DEB_CXXFLAGS_SET = "${params.DEB_CXXFLAGS_SET}"
        DEB_CFLAGS_SET = "${params.DEB_CFLAGS_SET}"
        REPO_URL = "${params.REPO_URL}"
    }
    stages {
        stage('Checkout') {
            steps {
                cleanWs()
                checkout([
                    $class: 'GitSCM',
                    branches: [[name: "${BRANCH}" ]],
                    userRemoteConfigs: [[url: "${REPO_URL}"]],
                    extensions: [[$class: 'RelativeTargetDirectory', relativeTargetDir: 'stellar-core']]
                ])
            }
        }
        stage('Build BUILD_TESTS') {
            steps {
                withEnv(["BUILD_WITH_TESTS=true"]) {
                    buildPackage()
                }
            }
        }
        stage('Build without BUILD_TESTS') {
            steps {
                withEnv(["BUILD_WITH_TESTS=false"]) {
                    buildPackage()
                }
            }
        }
    }
    post {
        success {
            build job: 'stellar-core-docker-pipeline/', wait: false, parameters: [string(name: 'VERSION', value: "${BUILD_VERSION}"), string(name: 'BUILDNUM', value: "${BUILD_NUMBER}")]
        }
    }
}

def buildPackage() {
    sh '''#!/bin/bash

        echo "info: building ${BRANCH}";

        # disable tests, enable parallel builds
        export DEB_BUILD_OPTIONS="nocheck parallel=$(($(nproc)+1))"

        if [ $BUILD_WITH_TESTS = "true" ]; 
        then
            export VERSION=$BUILD_VERSION~buildtests;
            export DEB_CONFIGURE_OPTS="$DEB_CONFIGURE_OPTS --enable-extra-checks";

        else
            export VERSION=$BUILD_VERSION;
            export DEB_CONFIGURE_OPTS="$DEB_CONFIGURE_OPTS --disable-tests";
        fi

        # These variables are used in src/Makefile.am
        export LOCAL_PACKAGE='stellar-core'; 
        export LOCAL_VERSION=$VERSION-$BUILD_NUMBER;

        env

        if [ -d stellar-core ]; then
            echo 'info: preparing code directory';
            cp -r stellar-core stellar-core-$VERSION;
            cd stellar-core-$VERSION;

            export GIT_COMMIT=$(git rev-parse HEAD) # used in src/Makefile.am

            echo 'info: updating git submodules';
            git submodule init;
            git submodule update;
            cd ${WORKSPACE};
            echo 'info: creating upstream tarball';
            tar czf stellar-core-$VERSION.tar.gz stellar-core-$VERSION/;
        fi

        # checkout debhelper files
        git clone git@github.com:stellar/packages.git;

        if [ -d packages ]; then
            echo "info: updating changelog to match $VERSION";
            sed -i "s/_VERSION_/$VERSION/" packages/stellar-core/debian/changelog;
            sed -i "s/_BUILD_VERSION_/$BUILD_NUMBER/" packages/stellar-core/debian/changelog;
            # THIS PACKAGE WILL END UP IN THE /PUBLIC UNSTABLE/ repo
            sed -i "s/) stable;/) unstable;/" packages/stellar-core/debian/changelog;
            # we want parallel builds
            sed -i "s/--with autoreconf --with systemd/--with autoreconf --with systemd --parallel/" packages/stellar-core/debian/rules;
            # we only need the stellar-core/debian dir
            cp -r packages/stellar-core/debian stellar-core-$VERSION/;
            # remove packages checkout
            rm -r packages/;
        fi

        cd stellar-core-$VERSION/;
        # create upstream tarball, etc
        dh_make -ysf ../stellar-core-$VERSION.tar.gz
        
        # use pbuilder for clean/reproducible builds
        if [ -f /var/cache/pbuilder/base.tgz ]; then
            echo 'info: updating base.tgz'
            sudo --preserve-env /usr/sbin/pbuilder update --debootstrapopts --variant=buildd --distribution xenial
        else
            echo 'info: creating base.tgz'
            sudo --preserve-env /usr/sbin/pbuilder create --debootstrapopts --variant=buildd --distribution xenial
        fi

        # build the package
        /usr/bin/pdebuild --buildresult /var/lib/repository/public/mini-dinstall/incoming/ --debbuildopts -b --logfile ${WORKSPACE}/pdebuild.log

        # cleanup incoming
        rm -f /var/lib/repository/public/mini-dinstall/incoming/{*.tar.xz,*.dsc,*.tar.gz}
    '''
}