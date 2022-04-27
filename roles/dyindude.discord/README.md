dyindude.discord [![CircleCI build status](https://circleci.com/gh/dyindude/ansible-role-discord/tree/master.svg?style=svg)](https://circleci.com/gh/dyindude/ansible-role-discord/tree/master)
=========

This role checks the current available version of discord for Linux from discordapp.com and installs or updates the package.

Role Variables
--------------

The only default variable used in this role is the URL provided to download the discord package. You should not need to change this unless the official path somehow changes.

`discord_package_url: "https://discordapp.com/api/download?platform=linux&format=deb"`

Example Playbook
----------------

    - hosts: laptop
      roles:
         - dyindude.discord

License
-------

BSD

Author Information
------------------

Created by github.com/dyindude
