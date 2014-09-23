gatewayd-installers
===================

Thoroughly tested installation scripts for gatewayd et al. In general, based on wiki and documentation already available. Provides gatewayd, ripple-rest and postgresql instructions. Also automates configuration to a large extent, including password generation and startup script creation.

These instructions *should* work for any Ubuntu distro >= 12.04, but we have only tested on 14.04.1

##Notes
**This is a work in progress**
  - Currently, gatewayd cannot be configured to use ripple-rest over https. "SSL" in gatewayd/config/config.js must be set to false. The ripple-rest api, however, works fine. I believe this is MY problem and does not reflect an issue in gatewayd.

##Thanks
  - @jzlcdh for exhaustive testing and an invaluable stream of useful, productive feedback.
