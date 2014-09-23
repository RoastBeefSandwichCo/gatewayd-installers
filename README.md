gatewayd-installers
===================

Thoroughly tested installation scripts for gatewayd et al. In general, based on wiki and documentation already available. Provides gatewayd, ripple-rest and postgresql instructions. Also automates configuration to a large extent, including password generation and startup script creation.

These instructions *should* work for any Ubuntu distro >= 12.04, but we have only tested on 14.04.1

##Notes
**This is a work in progress**
  - Currently, postgresql https is not working. You must remove/comment the ssl lines in ripple-rest/config.json

##Thanks
  - @jzlcdh for exhaustive testing and an invaluable stream of useful, productive feedback.
