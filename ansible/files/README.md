# ansible/files/

`tuya-power` (the smart-strip PDU script) belongs here so the playbook can
install it to `/usr/local/bin/tuya-power` (mode 0700, root) — but it embeds
the Tuya cloud ACCESS_ID/ACCESS_SECRET, so it is gitignored and must never
be committed. Copy your working script from the host into this directory
before running the playbook:

    scp ali@192.168.101.204:/usr/local/bin/tuya-power ansible/files/

If the file is absent the playbook simply skips that task. The script's
Python venv (`/opt/tuya-env`) is a one-time manual install per the
IT-provided instructions; the Tuya IoT Core trial subscription also expires
periodically and needs renewing in the Tuya console when `tuya-power`
starts failing.
