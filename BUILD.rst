Compiling the Linuxfabrik Monitoring Plugins
============================================

HOWTO build / compile / make them and put them into Linux packages like rpm or deb or exe for Windows.


Compiling for Linux
-------------------

.. note::

    You can find the pre-built OS packages at https://repo.linuxfabrik.ch/monitoring-plugins/.


Introduction
~~~~~~~~~~~~

The Linuxfabrik Monitoring Plugins are compiled with ``pyinstaller`` while the OS installation packages are built afterwards with `FPM <https://docs.linuxfabrik.ch/software/fpm.html>`_. This allows us to completely avoid Python on the target systems.

What this HOWTO covers:

* rpm for RHEL 7+
* deb for Debian 10+, Ubuntu 18+
* tar
* zip

Currently not implemented:

* apk (= Alpine; because of errors while running ``fpm -s deb -t apk linuxfabrik-monitoring-plugins.deb`` and ``fpm -s rpm -t apk linuxfabrik-monitoring-plugins.rpm``)
* freebsd (because of errors while running ``fpm -s deb -t freebsd linuxfabrik-monitoring-plugins.deb`` and ``fpm -s rpm -t freebsd linuxfabrik-monitoring-plugins.rpm``)
* pacman (= Arch Linux; because of errors while running ``fpm -s deb -t pacman linuxfabrik-monitoring-plugins.deb``, ``fpm -s rpm -t pacman linuxfabrik-monitoring-plugins.rpm`` and ``fpm -t pacman -p linuxfabrik-monitoring-plugins.pkg.tar.zst``)

Not tested:

* OpenSUSE

Run everything under root (``sudo su -`` or ``sudo -i``).


Prepare Build Environment
~~~~~~~~~~~~~~~~~~~~~~~~~

Important: Use Python 3.8+.

RHEL 7
    .. code-block:: bash

        yum -y update
        yum -y install git
        yum -y install binutils

        yum -y install centos-release-scl

        # use latest python available from scl
        yum -y install rh-python38 rh-python38-python-devel
        scl enable rh-python38 bash

        # use latest ruby available from scl
        yum -y install gcc redhat-rpm-config rpm-build squashfs-tools
        yum -y install rh-ruby30 rh-ruby30-ruby-devel
        scl enable rh-ruby30 bash

RHEL 8
    .. code-block:: bash

        yum -y update
        yum -y install git zip
        yum -y install binutils

        yum -y install python39 python39-devel
        alias python3=python3.9

        yum -y install ruby-devel gcc make rpm-build libffi-devel

RHEL 9
    .. code-block:: bash

        yum -y update
        yum -y install git zip
        yum -y install binutils

        yum -y install ruby-devel gcc make rpm-build libffi-devel

Debian 10
    .. code-block:: bash

        apt-get -y update
        apt-get -y install git
        apt-get -y install python3-venv python3-pip

        apt-get install -y ruby ruby-dev rubygems build-essential

Debian 11
    .. code-block:: bash

        apt-get -y update
        apt-get -y install git
        apt-get -y install python3-venv

        apt-get install -y ruby ruby-dev rubygems build-essential

Ubuntu 18
    .. code-block:: bash

        apt-get -y update
        apt-get -y install git
        apt-get -y install binutils
        apt-get -y install python3-pip python3-venv

        apt-get install -y ruby ruby-dev rubygems build-essential

Ubuntu 20
    .. code-block:: bash

        apt-get -y update
        apt-get -y install git
        apt-get -y install python3-venv

        apt-get install -y ruby ruby-dev rubygems build-essential

Ubuntu 22
    .. code-block:: bash

        apt-get -y update
        apt-get -y install git
        apt-get -y install python3-venv

        apt-get install -y ruby ruby-dev rubygems build-essential

Now FPM can be installed:

.. code-block:: bash

    # install fpm using gem
    gem install fpm


Compile
~~~~~~~

01: Create Python Env
    .. code-block:: bash

        python3 -m venv --system-site-packages pyinstaller
        source pyinstaller/bin/activate

        pip install --upgrade pip

        pip install --upgrade wheel
        pip install --upgrade setuptools
        pip install pyinstaller

        # install any libraries specific for the project:
        pip install argparse
        pip install beautifulsoup4
        pip install certifi
        pip install cffi
        pip install colorama
        pip install counter
        pip install datetime
        pip install jinja2
        pip install lxml
        pip install netifaces
        pip install path
        pip install psutil
        pip install pymysql
        pip install pysmbclient
        pip install python-keystoneclient
        pip install python-swiftclient
        pip install smbprotocol
        pip install uuid
        pip install vici
        pip install xmltodict

02: git clone, checkout
    .. code-block:: bash

        RELEASE=2023030801 # has to start with a digit
        RELEASE_TYPE='release' # or 'testing'

        git clone https://github.com/Linuxfabrik/monitoring-plugins.git
        git clone https://github.com/Linuxfabrik/lib.git

        cd monitoring-plugins
        git checkout $RELEASE
        # note that this will not work when using a commit hash, in that case manually checkout the correct version
        cd ..

        cd lib
        git checkout $RELEASE
        cd ..

03: Create compile script
    .. code-block:: bash

        cat > make << 'EOF'
        #!/usr/bin/env bash

        # cleanup old files
        rm -rf /tmp/dist
        mkdir -p /tmp/dist/summary/{check,notification}-plugins

        for dir in monitoring-plugins/check-plugins/*; do
            check="$(basename $dir)"
            if [ "$check" != "example" ]; then
                echo -e "\ncompiling $check..."
                pyinstaller --clean --distpath /tmp/dist/check-plugins --workpath /tmp/build/check-plugins --specpath /tmp/spec/check-plugins --noconfirm --noupx --onedir "$dir/${check}"
            fi
        done
        \cp -a /tmp/dist/check-plugins/*/* /tmp/dist/summary/check-plugins

        for dir in monitoring-plugins/notification-plugins/*; do
            notification="$(basename $dir)"
            if [ "$notification" != "example" ]; then
                echo -e "\ncompiling $notification..."
                pyinstaller --clean --distpath /tmp/dist/notification-plugins --workpath /tmp/build/notification-plugins --specpath /tmp/spec/notification-plugins --noconfirm --noupx --onedir "$dir/${notification}"
            fi
        done
        \cp -a /tmp/dist/notification-plugins/*/* /tmp/dist/summary/notification-plugins
        EOF

04: Compile
    .. code-block:: bash

        # takes round about 10 minutes
        chmod +x make
        ./make


Build OS Packages
~~~~~~~~~~~~~~~~~

Here, ``fpm`` creates the package names on its own.

Create the ``.fpm`` config file:

.. code-block:: bash

    mkdir -p check-plugins
    cd check-plugins

    # script to be run after package installation
    cat > rpm-post-install << EOF
    restorecon -r /usr/lib64/nagios
    setsebool -P nagios_run_sudo on
    EOF

    cat > .fpm << EOF
    --after-install rpm-post-install
    --architecture all
    --chdir /tmp/dist/summary/check-plugins
    --description "This Enterprise Class Check Plugin Collection offers a bunch of Nagios-compatible check plugins for Icinga, Naemon, Nagios, OP5, Shinken, Sensu and other monitoring applications. Each plugin is a stand-alone command line tool that provides a specific type of check. Typically, your monitoring software will run these check plugins to determine the current status of hosts and services on your network."
    --input-type dir
    --license "The Unlicense"
    --maintainer "info@linuxfabrik.ch"
    --name linuxfabrik-monitoring-plugins
    --rpm-summary "The Linuxfabrik Monitoring Plugins Collection (Check Plugins)"
    --url "https://github.com/Linuxfabrik/monitoring-plugins"
    --vendor "Linuxfabrik GmbH, Zurich, Switzerland"
    --version $RELEASE
    EOF

    for file in $(cd /tmp/dist/summary/check-plugins; find . -type f | sort); do
        # strip leading './'
        file="${file#./}"
        echo "$file=/usr/lib64/nagios/plugins/$file" >> .fpm
    done

    cd ..

.. code-block:: bash

    mkdir -p notification-plugins
    cd notification-plugins

    cat > .fpm << EOF
    --after-install rpm-post-install
    --architecture all
    --chdir /tmp/dist/summary/notification-plugins
    --description "Notification scripts for Icinga."
    --input-type dir
    --license "The Unlicense"
    --maintainer "info@linuxfabrik.ch"
    --name linuxfabrik-notification-plugins
    --rpm-summary "The Linuxfabrik Monitoring Plugins Collection (Notification Plugins)"
    --url "https://github.com/Linuxfabrik/monitoring-plugins"
    --vendor "Linuxfabrik GmbH, Zurich, Switzerland"
    --version $RELEASE
    EOF

    for file in $(cd /tmp/dist/summary/notification-plugins; find . -type f | sort); do
        # strip leading './'
        file="${file#./}"
        echo "$file=/usr/lib64/nagios/plugins/$file" >> .fpm
    done

    cd ..

Create the OS packages. Important: Be sure to build the binaries for the ``.tar`` and ``.zip`` file on RHEL 7, otherwise there will be `problems because of a too new linked glibc <https://github.com/Linuxfabrik/monitoring-plugins/issues/661>`_ if these binaries are used on older systems:

* RHEL 7: Glibc 2.17
* Debian 10: Glibc 2.28
* RHEL 8: Glibc 2.28
* Debian 11: Glibc 2.31
* RHEL 9: Glibc 2.34
* Debian 12: Glibc 2.36

RHEL 7
    .. code-block:: bash

        cd check-plugins
        fpm --output-type rpm
        fpm --output-type tar
        fpm --output-type zip
        cd ..

        cd notification-plugins
        fpm --output-type rpm
        fpm --output-type tar
        fpm --output-type zip
        cd ..

RHEL 8+
    .. code-block:: bash

        cd check-plugins
        fpm --output-type rpm
        cd ..

        cd notification-plugins
        fpm --output-type rpm
        cd ..

Debian 10+ / Ubuntu 18+
    .. code-block:: bash

        cd check-plugins
        fpm --output-type deb
        cd ..

        cd notification-plugins
        fpm --output-type deb
        cd ..


Compiling for Windows
---------------------

To allow running the check plugins under Windows without installing Python, compile the check plugins using `Nuitka <https://nuitka.net/>`_. For this, you need a Windows Machine with Python 3 and Nutika installed (see the `official installation guide <https://nuitka.net/doc/user-manual.html#installation>`_, we recommend using ``pip`` for simplicity).

Use the `Linuxfabrik LFOps "monitoring-plugins" role <https://github.com/Linuxfabrik/lfops/tree/main/roles/monitoring_plugins>`_:

.. code-block:: bash

    ansible-playbook \
        --inventory=inventory \
        --tags=monitoring_plugins,monitoring_plugins:nuitka_compile \
        --extra-vars='monitoring_plugins__windows_variant=python monitoring_plugins__repo_version=main' \
        --limit=windows-machine \
        linuxfabrik.lfops.monitoring_plugins

To let the Ansible role know which check-plugin to compile for Windows, create an empty ``.windows`` file in the check-plugin folder.

Then copy ``C:\\nuitka-compile-temp`` to a Linux Machine and zip it:

.. code-block:: bash

    cd /path/to/nuitka-compile-temp
    mkdir output
    for dir in */; do
        echo $dir
        file=$(basename $dir | sed 's/.dist//')
        cp -rv $dir* output
    done

    cd output
    zip -r ../monitoring-plugins.zip .

Rename the ``monitoring-plugins.zip`` to the correct version.


