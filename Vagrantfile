# -*- mode: ruby;  ruby-indent-tabs-mode: t -*-
# vi: set ft=ruby :
# **************************************************************************
# Add specific configuration for running IPython notebooks on a VM
# **************************************************************************

# --------------------------------------------------------------------------
# Variables defining the configuration of notebook 
# Modify as needed

# RAM memory used for the VM, in MB
vm_memory = '2048'
# Number of CPU cores assigned to the VM
vm_cpus = '1'

# Password to use to access the Notebook web interface 
vm_password = 'vmuser'

# Username that will run all processes.
vm_username = 'vmuser'

# The virtual machine exports the port where the notebook process by forwarding
# it to this port of the local machine
# So to access the notebook server, you point to http://localhost:<port>
port_nb = 8008


# --------------------------------------------------------------------------
# Some variables that affect Vagrant execution

# Check the command requested -- if ssh we'll change the login user
vagrant_command = ARGV[0]

# Conditionally activate some provision sections
provision_run_rs  = ENV['PROVISION_RSTUDIO'] == '1' || \
        (vagrant_command == 'provision' && ARGV.include?('rstudio'))
provision_run_nbc = (ENV['PROVISION_NBC'] == '1') || \
        (vagrant_command == 'provision' && \
           (ARGV.include?('nbc')||ARGV.include?('nbc.es')))
provision_run_ai  = ENV['PROVISION_AI'] == '1' || \
        (vagrant_command == 'provision' && ARGV.include?('ai'))
provision_run_mvn = ENV['PROVISION_MVN'] == '1' || \
        (vagrant_command == 'provision' && ARGV.include?('mvn'))
provision_run_scaladev = ENV['PROVISION_SBT'] == '1' || \
        (vagrant_command == 'provision' && ARGV.include?('scaladev'))
provision_run_dl  = ENV['PROVISION_DL'] == '1' || \
        (vagrant_command == 'provision' && ARGV.include?('dl'))

#provision_run_rs = true
#provision_run_ai = true


# --------------------------------------------------------------------------
# Vagrant configuration

port_nb_internal = 8008

# The "2" in Vagrant.configure sets the configuration version
Vagrant.configure(2) do |config|

  # This is to avoid Vagrant inserting a new SSH key, instead of the
  # default one (perhaps because the box will be later packaged)
  #config.ssh.insert_key = false

  # Use our custom username, instead of the default "vagrant"
  if vagrant_command == "ssh"
      config.ssh.username = vm_username
  end
  #config.ssh.username = "vagrant"


  config.vm.define "vm-ml-nb64" do |vgrml|

    # The base box we are using. As fetched from ATLAS
    vgrml.vm.box = "paulovn/ml-base64"
    vgrml.vm.box_version = "= 0.9.0"
    # Alternative place: box elsewhere
    #vgrml.vm.box_url = "http://tiny.cc/ml-base64-090-box"

    # Disable automatic box update checking. If you disable this, then
    # boxes will only be checked for updates when the user runs
    # `vagrant box outdated`. This is not recommended.
    # vgrml.vm.box_check_update = false

    # Deactivate the usual synced folder and use instead a local subdirectory
    vgrml.vm.synced_folder ".", "/vagrant", disabled: true
    vgrml.vm.synced_folder "vmfiles", "/vagrant", 
      mount_options: ["dmode=775","fmode=664"],
      disabled: false
    #owner: vm_username
    #auto_mount: false
  
    # Customize the virtual machine: set hostname & allocated RAM
    vgrml.vm.hostname = "vm-ipnb-ml"
    vgrml.vm.provider :virtualbox do |vb|
      # Set the hostname in VirtualBox
      vb.name = vgrml.vm.hostname.to_s
      # Customize the amount of memory on the VM
      vb.memory = vm_memory
      # Set the number of CPUs
      vb.cpus = vm_cpus
      # Display the VirtualBox GUI when booting the machine
      #vb.gui = true
    end

    # **********************************************************************
    # Networking

    # ---- NAT interface ----
    # NAT port forwarding
    vgrml.vm.network :forwarded_port, 
     #auto_correct: true,
     guest: port_nb_internal,
     host: port_nb                  # Notebook UI

    # RStudio server
    # =====> uncomment if using RStudio
    #vgrml.vm.network :forwarded_port, host: 8787, guest: 8787

    # Quiver
    # =====> uncomment if using Quiver visualization for Keras
    #vgrml.vm.network :forwarded_port, host: 5000, guest: 5000

    # ---- bridged interface ----
    # Declare a public network
    # This enables the machine to be connected from outside
    # =====> Uncomment the following two lines to enable bridge mode:
    #vgrml.vm.network "public_network",
    #type: "dhcp"

    # ===> if the host has more than one interface, we can set which one to use
    #bridge: "wlan0"
    # ===> we can also set the MAC address we will send to the DHCP server
    #:mac => "08002710A7ED"


    # ---- private interface ----
    # Create a private network, which allows host-only access to the machine
    # using a specific IP.
    #vgrml.vm.network "private_network", ip: "192.72.33.10"


    vgrml.vm.post_up_message = "**** The Vagrant ML-Notebook machine is up. Connect to http://localhost:" + port_nb.to_s


    # **********************************************************************
    # Provisioning: install configuration files and startup scripts

    # .........................................
    # Create the user to run jobs (esp. notebook processes)
    vgrml.vm.provision "01.nbuser",
    type: "shell", 
    privileged: true,
    args: [ vm_username ],
    inline: <<-SHELL
      id "$1" >/dev/null 2>&1 || useradd -c 'User for Notebook' -m -G vagrant "$1"

      # Create the .bash_profile file
      cat <<'ENDPROFILE' > /home/$1/.bash_profile
# .bash_profile

# Get the aliases and functions
if [ -f ~/.bashrc ]; then
   . ~/.bashrc
fi

# User specific environment and startup programs
export PATH=$HOME/bin:$PATH:$HOME/.local/bin
ENDPROFILE
      chown $1.$1 /home/$1/.bash_profile

      # Create some local files as the designated user
      su -l "$1" <<'USEREOF'
for d in bin tmp .ssh .jupyter .Rlibrary; do test -d $d || mkdir $d; done
chmod 700 .ssh
rm -f bin/{python,python2.7,pip,ipython,jupyter}
ln -s /opt/ipnb/bin/{python,python2.7,pip,ipython,jupyter} bin
test -h IPNB || { rm -f IPNB; ln -s /vagrant/IPNB/ IPNB; }
cat <<'BASHRC' >> .bashrc
# Place where to keep user R packages
export R_LIBS_USER=~/.Rlibrary
# Load Theano initialization file
export THEANORC=/etc/theanorc:~/.theanorc
# Jupyter uses this to define datadir but it is undefined when using "runuser"
test "$XDG_RUNTIME_DIR" || export XDG_RUNTIME_DIR=/run/user/$(id -u)
BASHRC
USEREOF

      # Install the vagrant public key so that we can ssh to this account
      cp -p /home/vagrant/.ssh/authorized_keys /home/$1/.ssh/authorized_keys
      chown $1.$1 /home/$1/.ssh/authorized_keys
      
    SHELL

    # Mount the shared folder with the new created user, so that it can write
    # ---> don't, instead we add the user to the vagrant group and mount the 
    #      shared folder with group permissions
#    vgrml.vm.provision "02.mount",
#    type: "shell",
#    privileged: true,
#    keep_color: true,
#    args: [ vm_username ],
#    inline: <<-SHELL
#umount /vagrant
#mount -t vboxsf -o uid=$(id -u $1),gid=$(id -g $1) vagrant /vagrant
#SHELL

    # .........................................
    # Create the IPython Notebook profile 
    # and install IRKernel, and extensions
    # Prepared for IPython >=4 (so that we configure as a Jupyter app)
    vgrml.vm.provision "03.nbkernels",
    type: "shell", 
    privileged: true,
    keep_color: true,    
    args: [ vm_username, vm_password, port_nb_internal ],
    inline: <<-SHELL

     USERNAME=$1

     # --------------------- Create the Jupyter config
     echo "Creating Jupyter config"
     PWD=$(/opt/ipnb/bin/python -c "from IPython.lib import passwd; print passwd('$2')")
     cat <<-EOF > /home/$USERNAME/.jupyter/jupyter_notebook_config.py
c = get_config()
# define server
c.NotebookApp.ip = '*'
c.NotebookApp.port = $3
c.NotebookApp.password = u'$PWD'
c.NotebookApp.open_browser = False
c.NotebookApp.log_level = 'INFO'
c.NotebookApp.notebook_dir = u'/home/$USERNAME/IPNB'
# Preload matplotlib
c.IPKernelApp.matplotlib = 'inline'
# Kernel heartbeat interval in seconds.
# This is in jupyter_client.restarter. Not sure if it gets picked up
c.KernelRestarter.time_to_dead = 30.0
c.KernelRestarter.debug = True
EOF
     chown $USERNAME.$USERNAME /home/$USERNAME/.jupyter/jupyter_notebook_config.py

     KDIR=/home/$USERNAME/.local/share/jupyter/kernels

     # --------------------- Install the IRkernel
     echo "Installing IRkernel ..."
     su -l "$USERNAME" <<-EOF
PATH=/opt/ipnb/bin:$PATH LD_LIBRARY_PATH=/opt/rh/python27/root/usr/lib64 Rscript -e 'IRkernel::installspec()'
EOF

     # --------------------- Install the notebook extensions
     echo "Installing notebook extensions"
     su -l "$USERNAME" <<-EOF
python2.7 -c 'from notebook.services.config import ConfigManager; ConfigManager().update("notebook", {"load_extensions": {"toc": True, "toggle-headers": True, "search-replace": True, "python-markdown": True }})'
     ln -fs /opt/ipnb/share/jupyter/pre_pymarkdown.py /opt/ipnb/lib/python2.7/site-packages
EOF

     # --------------------- Put the custom Jupyter icon in place
     cd /opt/ipnb/lib/python2.7/site-packages/notebook/static/base/images
     mv favicon.ico favicon-orig.ico
     ln -s favicon-custom.ico favicon.ico

    SHELL

    # .........................................
    # Install the Notebook startup script & configure it
    vgrml.vm.provision "04.nbconfig",
    type: "shell", 
    privileged: true,
    keep_color: true,    
    args: [ vm_username ],
    inline: <<-SHELL
     # Link the IPython mgr script so that it can be found by root
     chmod 775 /opt/ipnb/bin/jupyter-notebook-mgr
     rm -f /usr/sbin/notebook
     ln -s /opt/ipnb/bin/jupyter-notebook-mgr /usr/sbin
     # note we do not enable the service -- we will explicitly start it at the end

     # Create the config for IPython notebook
     cat <<-EOF > /etc/sysconfig/jupyter-notebook
NOTEBOOK_USER="$1"
NOTEBOOK_SCRIPT="/opt/ipnb/bin/jupyter-notebook"
EOF

    SHELL

    # *************************************************************************
    # Optional packages

    # .........................................
    # Install the neuralnet R package
    # vgrml.vm.provision "neuralnet",
    # type: "shell",
    # keep_color: true,
    # privileged: true,
    # inline: <<-SHELL
    #  echo "Installing R packages"
    #  for pkg in '"neuralnet"'
    #  do
    #      echo -e "\nInstalling R packages: $pkg"
    #      Rscript -e "install.packages(c($pkg),dependencies=TRUE,repos=c('http://ftp.cixug.es/CRAN/','http://cran.es.r-project.org/'),quiet=FALSE)"
    #  done
    # SHELL

    # .........................................
    # Install RStudio server
    # Do it only if explicitly requested (either by environment variable 
    # PROVISION_RSTUDIO when creating or by --provision-with rstudio) 
    # *** Don't forget to also uncomment forwarding for port 8787!
    if (provision_run_rs)
      vgrml.vm.provision "rstudio",
      type: "shell",
      keep_color: true,
      privileged: true,
      args: [ vm_username, vm_password ],
      inline: <<-SHELL
        echo "Downloading & installing RStudio Server"
        # Download & install the RPM for RStudio server
        PKG=rstudio-server-rhel-1.0.136-x86_64.rpm
        wget --no-verbose https://s3.amazonaws.com/rstudio-dailybuilds/$PKG
        yum install -y --nogpgcheck $PKG
        rm $PKG
        # Define the directory for the user library
        CNF=/etc/rstudio/rsession.conf
        grep -q r-libs-user $CNF || echo "r-libs-user=~/.Rlibrary" >> $CNF
        # Create a link to the host-mounted R subdirectory
        sudo -i -u "$1" bash -c "rm -f R; ln -s /vagrant/R/ R"
        # Set the password for the user, so that it can log in in RStudio
        echo "$2" | passwd --stdin "$1"
        # Send message
        echo "RStudio Server should be accessed at http://localhost:8787"
        echo "(if not, check in Vagrantfile that port 8787 has been forwarded)"
      SHELL
    end

    # .........................................
    # Install the necessary components for nbconvert to work.
    # Do it only if explicitly requested (either by environment variable 
    # PROVISION_NBC when creating or by --provision-with nbc)
    if (provision_run_nbc)
      vgrml.vm.provision "nbc",
      type: "shell",
      privileged: true,
      keep_color: true,
      args: [ vm_username ],
      inline: <<-SHELL
          echo "Installing nbconvert requirements"
          yum install -y pandoc inkscape texlive-xetex texlive-xetex-def
          sudo -i -u vagrant pip install pandoc
          DIR=$(kpsewhich -var-value TEXMFLOCAL)
          mkdir -p $DIR
          cd $DIR
          for p in collectbox adjustbox; do
            wget --no-verbose http://mirrors.ctan.org/install/macros/latex/contrib/$p.tds.zip
            unzip -o $p.tds.zip
            rm $p.tds.zip
          done
          # The LaTeX generated by nbconvert uses the ulem & upquote packages
          # But they are not available as tds install packages. And ulem
          # has a RPM package tex(ulem), but upquote does not
          cd tex/latex
          for p in ulem upquote; do
            wget --no-verbose http://mirrors.ctan.org/macros/latex/contrib/$p/$p.sty
          done
          texhash
          # Finally we modify the LaTeX template to generate A4 pages
          # (comment this out to keep Letter-sized pages)
          perl -pi -e 's|(\\\\geometry{)|${1}a4paper,|' /opt/ipnb/lib/python2.7/site-packages/nbconvert/templates/latex/base.tplx
      SHELL

      # .........................................
      # Optional: modify nbconvert to process Spanish documents
      vgrml.vm.provision "nbc.es",
      type: "shell",
      privileged: false,
      keep_color: true,
      inline: <<-SHELL
          LANGUAGE=spanish
          CODE=es
          echo "** Adding support for $LANGUAGE to LaTeX"
          # https://tex.stackexchange.com/questions/345632/f25-texlive2016-no-hyphenation-patterns-were-preloaded-for-the-language-russian
          sudo yum install -y texlive-polyglossia texlive-euenc texlive-hyph-utf8
          LANGDAT=$(kpsewhich language.dat)
          sudo bash -c "echo -e '\n$LANGUAGE hyph-${CODE}.tex\n=use$LANGUAGE' >> $LANGDAT" && sudo fmtutil-sys --all
          echo "** Converting base LaTeX template for $LANGUAGE"
          perl -pi -e 's(\\\\usepackage\\[T1\\]{fontenc})(\\\\usepackage{polyglossia}\\\\setmainlanguage{'$LANGUAGE'});' -e 's#\\\\usepackage\\[utf8x\\]{inputenc}#%--removed--#;' /opt/ipnb/lib/python2.7/site-packages/nbconvert/templates/latex/base.tplx
      SHELL

    end

    # .........................................
    # Install the additional packages for the AI course
    # Do it only if explicitly requested (either by environment variable 
    # PROVISION_AI when creating or by --provision-with ai) 
    if (provision_run_ai)
      vgrml.vm.provision "ai",
      type: "shell",
      privileged: true,
      keep_color: true,
      args: [ vm_username ],
      inline: <<-SHELL
        echo "Installing packages for AI course"
        yum -y install python27-tkinter
        su -l "vagrant" <<-EOF
         pip install nltk
         pip install aimlbotkernel
         pip install sparqlkernel
EOF
        su -l "$1" <<-EOF
         echo "Installing AIML-BOT & SPARQL kernels"
         jupyter aimlbotkernel install --user
         jupyter sparqlkernel install --user --logdir /var/log/ipnb
EOF

      SHELL
    end

    # .........................................
    # Install some Deep Learning stuff
    # Do it only if explicitly requested (either by environment variable 
    # PROVISION_DL when creating or by --provision-with dl) 
    if (provision_run_dl)
      vgrml.vm.provision "dl",
      type: "shell",
      privileged: false,
      keep_color: true,
      inline: <<-SHELL
         sudo yum install -y git
         #TF_BINARY=tensorflow-0.11.0-cp27-none-linux_x86_64.whl
         #URL=https://storage.googleapis.com/tensorflow/linux/cpu/$TF_BINARY
         #wget $URL
         #pip install --upgrade $TF_BINARY
         pip install --upgrade tensorflow
         pip install --upgrade --no-deps git+git://github.com/Theano/Theano.git
         pip install --upgrade keras quiver
         sudo yum remove -y git 
       SHELL
    end


    # *************************************************************************

    # .........................................
    # Start Jupyter Notebook
    # Note: this one we run it every time the machine boots, since during 
    # the VM boot sequence the startup script is executed before vagrant has 
    # mounted the shared folder, and hence it fails. 
    # Running it as a provisioning makes it run *after* vagrant mounts, so 
    # this way it works.
    # [An alternative would be to force mounting on startup, by adding the
    # vboxsf mount point to /etc/fstab during provisioning]
    vgrml.vm.provision "50.nbstart", 
      type: "shell", 
      run: "always",
      privileged: true,
      keep_color: true,    
      inline: "systemctl start notebook"


  end # config.vm.define

end
