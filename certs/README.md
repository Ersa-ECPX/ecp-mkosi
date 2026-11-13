# Ersa Compute Platform Secure Boot certificate

`ersa-secure-boot.der` and `ersa-secure-boot.pem` are the same **public** X.509 certificate in DER and PEM formats. They may be distributed with an Ersa Compute Platform image. The private signing key is stored separately and must never be distributed.

SHA-256 certificate fingerprint:

```
5C:A0:3F:12:44:CE:7C:96:EE:61:D5:EF:58:A7:4F:46:A2:0B:5F:CE:14:A6:97:EE:D6:72:FF:0D:B1:3D:B9:A9
```

Check the copy before enrollment:

```sh
openssl x509 -inform DER -in certs/ersa-secure-boot.der -noout -fingerprint -sha256
```

`mokutil --import certs/ersa-secure-boot.der` queues MOK enrollment on systems that boot through **shim**; complete it in MokManager after reboot. The current Ersa Compute Platform image boots systemd-boot directly, so MOK enrollment alone does not authorize its EFI binaries. That boot path requires firmware Secure Boot `db` trust for this certificate. Test enrollment in QEMU with disposable OVMF variables before changing physical firmware.
