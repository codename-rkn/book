# Run air-gapped

In order to run Apex Recon in an air-gapped environment you need to:

* Place the encrypted license file as `license` inside `~/.apex/` (or `$APEX_HOME/` if you've overridden the home dir).
    * Either by copying it over from a previous activation on an Internet-enabled machine — copy `~/.apex/license` (and, if present, the plaintext `~/.apex/license.key` next to it), or;
    * by [activating on-line](https://license.ecsypno.com/apex).
