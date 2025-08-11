# PKMN-WFC-Server Local Host Docker

Docker image based on [pkmn-wfc-server](https://github.com/u1f992/pkmn-wfc-server), [CoWFC](https://github.com/EnergyCube/CoWFC), and [Poké Classic Framework](https://github.com/mm201/pkmn-classic-framework).

> **Mystery Gifts are disabled** by default.

> **Since enabled weak-enough SSL auth on purpose, this container is not intended to be published via internet.**

> On 8/11/2025, this docker works.

## How to use

You need a Wi-Fi access point compatible with the NDS. Refer to guides for wiimmfi etc.

Also need to open ports listed [here](https://github.com/barronwaffles/dwc_network_server_emulator/wiki/Troubleshooting#port-forwarding).

0. Install WSL & Docker.Desktop
1. Edit `dnsmasq/wfc.conf`
2. Edit "args" in `docker-compose.yml`
3. Download [veekun-pokedex.sqlite.gz](https://veekun.com/static/pokedex/downloads/veekun-pokedex.sqlite.gz) and [wine-mono-7.4.0-x86.tar.xz](https://dl.winehq.org/wine/wine-mono/7.4.0/wine-mono-7.4.0-x86.tar.xz). Then move them to the same folder with docker-compose.yml
4. `docker-compose up`
5. Access the admin page `localhost/?page=admin&section=Dashboard`
6. Add your game to whitelist

### Game code (PKMN)

| title | code |
| --- | --- |
| Diamond | ADA |
| Pearl | APA |
| Platinum | CPU |
| HeartGold | IPK |
| SoulSilver | IPG |
| --- | --- |
| Black | IRB |
| White | IRA |
| Black2 | IRE |
| White2 | IRD |

> You can find other games' codes [here](https://www.cyberbotx.com:6883/Files).


## Reference

- [dwc_network_server_emulator](https://github.com/EnergyCube/dwc_network_server_emulator)
- [CoWFC](https://github.com/EnergyCube/CoWFC)
- [Poké Classic Framework](https://github.com/mm201/pkmn-classic-framework)
- [nds-constrain't](https://github.com/KaeruTeam/nds-constraint)
