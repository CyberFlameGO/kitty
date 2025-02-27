.. _shell_integration:

Shell integration
-------------------

kitty has the ability to integrate closely within common shells, such as `zsh
<https://www.zsh.org/>`_, `fish <https://fishshell.com>`_ and `bash
<https://www.gnu.org/software/bash/>`_ to enable features such as jumping to
previous prompts in the scrollback, viewing the output of the last command in
:program:`less`, using the mouse to move the cursor while editing prompts, etc.

.. versionadded:: 0.24.0

Features
-------------

* Open the output of the last command in a pager such as :program:`less`
  (:sc:`show_last_command_output`)

* Jump to the previous/next prompt in the scrollback
  (:sc:`scroll_to_previous_prompt` /  :sc:`scroll_to_next_prompt`)

* Click with the mouse anywhere in the current command to move the cursor there

* The current working directory or the command being executed are automatically
  displayed in the kitty window titlebar/tab title.

* The text cursor is changed to a bar when editing commands at the shell prompt

* Glitch free window resizing even with complex prompts. Achieved by erasing
  the prompt on resize and allowing the shell to redraw it cleanly.

* Sophisticated completion for the :program:`kitty` command in the shell.

* When confirming a quit command if a window is sitting at a shell prompt,
  it is not counted.


Configuration
---------------

Shell integration is controlled by :opt:`shell_integration`. By default, all
shell integration is enabled. Individual features can be turned off or it can
be disabled entirely as well. The :opt:`shell_integration` option takes a space
separated list of keywords:

disabled
    turn off all shell integration

no-rc
    dont modify the shell's rc files to enable integration. Useful if you prefer
    to :ref:`manually enable integration <manual_shell_integration>`.

no-cursor
    turn off changing of the text cursor to a bar when editing text

no-title
    turn off setting the kitty window/tab title based on shell state

no-prompt-mark
    turn off marking of prompts. This disables jumping to prompt, browsing
    output of last command and click to move cursor functionality.

no-complete
    turn off completion for the kitty command.


How it works
-----------------

At startup kitty detects if the shell you have configured (either system wide
or in kitty.conf) is a supported shell. If so, kitty injects some shell specific
code into the shell, to enable shell integration. How it does so varies for
different shells.


.. tab:: bash/zsh

   For these shells, kitty adds a couple of lines to
   the bottom of the shell's rc files (in an atomic manner) to load the shell
   integration code.

.. tab:: fish

    For fish, to make it automatically load the integration code provided by
    kitty, the integration script directory path is prepended to the
    :code:`XDG_DATA_DIRS` environment variable. This is only applied to the fish
    process and will be cleaned up by the integration script after startup. No files
    are added or modified.

Then, when launching the shell, kitty sets the environment variable
:envvar:`KITTY_SHELL_INTEGRATION` to the value of the :opt:`shell_integration`
option. The shell integration code reads the environment variable, turns on the
specified integration functionality and then unsets the variable so as to not
pollute the system. This has the nice effect that the changes to the shell's rc
files become no-ops when running the shell in anything other than kitty itself.

The actual shell integration code uses hooks provided by each shell to send
special escape codes to kitty, to perform the various tasks. You can see the
code used for each shell below:

.. raw:: html

    <details>
    <summary>Click to toggle shell integration code</summary>

.. tab:: zsh

    .. literalinclude:: ../shell-integration/kitty.zsh
        :language: zsh


.. tab:: fish

    .. literalinclude:: ../shell-integration/fish/vendor_conf.d/kitty-shell-integration.fish
        :language: fish

.. tab:: bash

    .. literalinclude:: ../shell-integration/kitty.bash
        :language: bash

.. raw:: html

   </details>


.. _manual_shell_integration:

Manual shell integration
----------------------------

If you do not want to rely on kitty's automatic shell integration or if you
want to setup shell integration for a remote system over SSH, in
:file:`kitty.conf` set:

.. code-block:: conf

    shell_integration disabled

Then in your shell's rc file, add the lines:

.. tab:: bash

    .. code-block:: sh

        if [[ ! -z "$KITTY_INSTALLATION_DIR" ]]; then
            export KITTY_SHELL_INTEGRATION="enabled"
            source "$KITTY_INSTALLATION_DIR/shell-integration/kitty.bash"
        fi

.. tab:: zsh

    .. code-block:: sh

        if [[ ! -z "$KITTY_INSTALLATION_DIR" ]]; then
            export KITTY_SHELL_INTEGRATION="enabled"
            source "$KITTY_INSTALLATION_DIR/shell-integration/kitty.zsh"
        fi

.. tab:: fish

    .. code-block:: fish

        if set -q KITTY_INSTALLATION_DIR
            set --global KITTY_SHELL_INTEGRATION enabled
            source "$KITTY_INSTALLATION_DIR/shell-integration/fish/vendor_conf.d/kitty-shell-integration.fish"
            set --prepend fish_complete_path "$KITTY_INSTALLATION_DIR/shell-integration/fish/vendor_completions.d"
        end


The value of :envvar:`KITTY_SHELL_INTEGRATION` is the same as that for
:opt:`shell_integration`, except if you want to disable shell integration
completely, in which case simply do not set the
:envvar:`KITTY_SHELL_INTEGRATION` variable at all.

If you want this to work while SSHing into a remote system, then you will
need to add some code to the snippets above to check if :code:`KITTY_INSTALLATION_DIR`
is empty and if so to set it to some hard coded location with the shell
integration scripts that need to be copied onto the remote system.


Notes for shell developers
-----------------------------

The protocol used for marking the prompt is very simple. You should consider
adding it to your shell as a builtin. Many modern terminals make use of it, for
example: kitty, iTerm2, WezTerm, DomTerm

Just before starting to draw the PS1 prompt send the escape code::

    <OSC>133;A<ST>

Just before starting to draw the PS2 prompt send the escape code::

    <OSC>133;A;k=s<ST>

Just before running a command/program, send the escape code::

    <OSC>133;C<ST>

Here ``<OSC>`` is the bytes ``0x1b 0x5d`` and ``<ST>`` is the bytes ``0x1b
0x5c``. This is exactly what is needed for shell integration in kitty. For the
full protocol, that also marks the command region, see `the iTerm2 docs
<https://iterm2.com/documentation-escape-codes.html>`_.
