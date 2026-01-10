Arch Linux
##########

Install the Task runner:

.. code-block:: bash

   sudo pacman -S go-task

Create the workspace directory and clone the repository:

.. code-block:: bash

   mkdir Thermoquad
   cd Thermoquad
   git clone git@github.com:Thermoquad/origin.git
   cd origin

Install system packages:

.. code-block:: bash

   task system-install:arch-linux

Configure your shell for direnv and asdf.

For ``.bashrc``:

.. code-block:: bash

   # asdf
   export ASDF_DATA_DIR="$HOME/.local/asdf"
   export PATH="$ASDF_DATA_DIR/shims:$PATH"

   # direnv
   eval "$(direnv hook bash)"

For ``.zshrc``:

.. code-block:: zsh

   # asdf
   export ASDF_DATA_DIR="$HOME/.local/asdf"
   export PATH="$ASDF_DATA_DIR/shims:$PATH"
   fpath=(${ASDF_DATA_DIR:-$HOME/.asdf}/completions $fpath)

   # direnv
   eval "$(direnv hook zsh)"

Restart your shell, then setup the workspace:

.. code-block:: bash

   cd Thermoquad/origin
   task setup-workspace

Leave and re-enter the directory to activate the environment, then install the SDKs:

.. code-block:: bash

   cd .. && cd origin
   task install-workspace
