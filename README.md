gatewayd-installers
===================

Thoroughly tested installation scripts for gatewayd et al. In general, based on wiki and documentation already available.

##Usage

Just open the txt or md corresponding to your distro. Only 14.04 is thoroughly documented atm.

##Notes
**This is a work in progress**
  - Ubuntu 14.04 Server
    - Issues 1, 4 for things you MUST address or your installation will not work (also mentioned in the txt)
    - You need the ssl script.
    - Currently, postgresql https is not working. You must remove/comment the ssl lines in ripple-rest/config.json

