# saphnet-komodo
The main resource sync to use for Komodo in the Sapphic Homelab/Home Server

Note that this is intended to work in tandem with [saphnet-compose-configs](https://github.com/AnarchoBooleanism/saphnet-compose-configs), and be initially introduced to host systems in [saphnet-nixos-configs](https://github.com/AnarchoBooleanism/saphnet-nixos-configs).

TODO:
- this repo separate from saphnet-compose-configs to keep separation of concerns separate between stacks and komodo, however stack resources are defined in saphnet-compose-configs, since stacks are very coupled to compose configs, and it's better to leave them in sync
- add note on how sops keys should be configured for each periphery config, also do this for saphnet-compose-configs README