# Directional scaling

This fork of YeetMouse adds four per-direction sensitivity multipliers. They
compensate for a sensor or a hand that tracks slightly differently depending on
the direction of the movement.

| Module parameter | Config key     | config.h define   | Applies to raw input moving |
|------------------|----------------|-------------------|-----------------------------|
| `DirScaleXPos`   | `dirScaleXPos` | `DIR_SCALE_X_POS` | right (x > 0)               |
| `DirScaleXNeg`   | `dirScaleXNeg` | `DIR_SCALE_X_NEG` | left (x < 0)                |
| `DirScaleYPos`   | `dirScaleYPos` | `DIR_SCALE_Y_POS` | down (y > 0)                |
| `DirScaleYNeg`   | `dirScaleYNeg` | `DIR_SCALE_Y_NEG` | up (y < 0)                  |

The multipliers are applied to the raw delta of every report, before the
Pre-Scale, so the speed calculation, the acceleration curve, rotation and angle
snapping all work on the scaled values. The default is `1` for all four, which
gives exactly the output of the unmodified driver.

## Setting the values

The values are plain decimal numbers with a dot as the separator (`0.9914`,
no exponent notation). There are three ways to set them:

- GUI: the "Directional Scale" inputs in the parameters panel, then "Apply"
  or "Apply + Save".
- Config file: add the four keys to `/etc/yeetmouse.conf` (`key=value`, one per
  line) and run `yeetmousectl apply /etc/yeetmouse.conf`.
- Directly through sysfs:

      echo 0.9914 > /sys/module/yeetmouse/parameters/DirScaleXPos
      echo 1 > /sys/module/yeetmouse/parameters/update

  The driver only reads the new values after `1` is written to `update`, on
  the next mouse movement and at most once per second.

## Persisting the values

Persistence works the same way as for every other parameter:

- `yeetmousectl save /etc/yeetmouse.conf` writes the values currently set in
  the driver to the config file. The file belongs to root, the GUI runs this
  through `pkexec` ("Apply + Save").
- `yeetmouse.service` runs `yeetmousectl apply /etc/yeetmouse.conf` at boot.

A config file without the four keys is still valid, the scales stay at `1`.
The unmodified yeetmousectl ignores the keys.

To let the members of the `yeetmouse` group save without root:

    sudo chgrp yeetmouse /etc/yeetmouse.conf
    sudo chmod 664 /etc/yeetmouse.conf

## Giving a user access

The parameters in `/sys/module/yeetmouse/parameters/` have the mode `0660`.
`yeetmouse.service` changes their group to `yeetmouse` every time it starts,
this covers all parameters, the new ones included.

    sudo usermod -aG yeetmouse <user>

The membership only applies to sessions started after this, so log out and in
again (or use `newgrp yeetmouse` in a single shell). Check with `id -nG`.

The parameters are created again as `root:root` whenever the module is loaded.
After reloading it by hand run `sudo systemctl restart yeetmouse.service`.

## Installing through DKMS

The fork uses the DKMS version `0.9.5+dirscale1`, the module name stays
`yeetmouse-driver`. On a machine without YeetMouse, `./install.sh` works as
usual.

On Arch based systems the pacman hook builds every module it finds in
`/usr/src` on a kernel update, so only one version of the source may be there.
To replace an installed 0.9.5, run from the root of this repository:

    make yeetmousectl GUI
    sudo make setup_dkms
    sudo dkms remove yeetmouse-driver/0.9.5 --all
    sudo mv /usr/src/yeetmouse-driver-0.9.5 /usr/local/src/
    sudo dkms install yeetmouse-driver/0.9.5+dirscale1
    sudo make install_userspace install_gui_files
    sudo yeetmousectl save /etc/yeetmouse.conf
    sudo rmmod yeetmouse
    sudo modprobe yeetmouse
    sudo systemctl restart yeetmouse.service

The `save` line keeps the values of the currently loaded driver. Leave it out
if `/etc/yeetmouse.conf` already has the values you want.

## Going back to the unmodified driver

    sudo dkms remove yeetmouse-driver/0.9.5+dirscale1 --all
    sudo rm -rf /usr/src/yeetmouse-driver-0.9.5+dirscale1
    sudo mv /usr/local/src/yeetmouse-driver-0.9.5 /usr/src/
    sudo dkms install yeetmouse-driver/0.9.5
    sudo rmmod yeetmouse
    sudo modprobe yeetmouse
    sudo systemctl restart yeetmouse.service

The GUI and yeetmousectl of the fork also work with the unmodified driver, the
directional scales are then simply ignored. `sudo yeetmouse-uninstall` removes
everything, whichever version is installed.
