Remember, this is meant to be run in a dockerized environment in VSCode, so open it there in a container

You can build this from the root directory of this repository
To build for the XIAO nRF52840 (richkbd_wireless): `west build -p -b seeeduino_xiao_ble app -- -DSHIELD=richkbd`


The output will be the .uf2 file which you can copy onto the keyboard when it is in bootloader mode.

If the keyboard is working, you can do that by pressing Nav - TOPLEFTMOSTKEY which will cause to mount as the
drive /Volumes/XIAO-SENSE

Then:
`cp -X build/zephyr/zmk.uf2 /Volumes/XIAO-SENSE`
For some reason this usually gives this error "cp: /Volumes/XIAO-SENSE/zmk.uf2: fcopyfile failed: Input/output error"
