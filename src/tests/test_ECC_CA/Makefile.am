dist_noinst_DATA = \
    SSSD_test_ECC_CA.config \
    SSSD_test_ECC_CA_key.pem \
    SSSD_test_ECC_cert_0001.config  \
    SSSD_test_ECC_cert_key_0001.pem

openssl_ecc_ca_config = $(srcdir)/SSSD_test_ECC_CA.config
openssl_ecc_ca_key = $(srcdir)/SSSD_test_ECC_CA_key.pem
pwdfile = pwdfile

configs := $(notdir $(wildcard $(srcdir)/SSSD_test_ECC_cert_*.config))
ids := $(subst SSSD_test_ECC_cert_,,$(basename $(configs)))
certs = $(addprefix SSSD_test_ECC_cert_x509_,$(addsuffix .pem,$(ids)))
certs_h = $(addprefix SSSD_test_ECC_cert_x509_,$(addsuffix .h,$(ids)))
pubkeys = $(addprefix SSSD_test_ECC_cert_pubsshkey_,$(addsuffix .pub,$(ids)))
pubkeys_h = $(addprefix SSSD_test_ECC_cert_pubsshkey_,$(addsuffix .h,$(ids)))
pkcs12 = $(addprefix SSSD_test_ECC_cert_pkcs12_,$(addsuffix .pem,$(ids)))

extra = softhsm2_ecc_one SSSD_test_ECC_crl.pem

# If openssl is run in parallel there might be conflicts with the serial
.NOTPARALLEL:

ca_all: clean serial SSSD_test_ECC_crl.pem $(certs) $(certs_h) $(pubkeys) $(pubkeys_h) $(pkcs12) $(extra)

$(pwdfile):
	@echo "123456" > $@

SSSD_test_ECC_CA.pem: $(openssl_ecc_ca_key) $(openssl_ecc_ca_config) serial
	$(OPENSSL) req -batch -config ${openssl_ecc_ca_config} -x509 -new -nodes -key $< -sha384 -days 1024 -set_serial 0 -extensions v3_ca -out $@


SSSD_test_ECC_cert_req_%.pem: $(srcdir)/SSSD_test_ECC_cert_key_%.pem $(srcdir)/SSSD_test_ECC_cert_%.config
	$(OPENSSL) req -new -nodes -key $< -reqexts req_exts -config $(srcdir)/SSSD_test_ECC_cert_$*.config -out $@

SSSD_test_ECC_cert_x509_%.pem: SSSD_test_ECC_cert_req_%.pem $(openssl_ecc_ca_config) SSSD_test_ECC_CA.pem
	$(OPENSSL) ca -config ${openssl_ecc_ca_config} -batch -notext -keyfile $(openssl_ecc_ca_key) -in $< -days 200 -extensions usr_cert -out $@

SSSD_test_ECC_cert_pkcs12_%.pem: SSSD_test_ECC_cert_x509_%.pem $(srcdir)/SSSD_test_ECC_cert_key_%.pem $(pwdfile)
	$(OPENSSL) pkcs12 -export -in SSSD_test_ECC_cert_x509_$*.pem -inkey $(srcdir)/SSSD_test_ECC_cert_key_$*.pem -nodes -passout file:$(pwdfile) -out $@

SSSD_test_ECC_cert_pubkey_%.pem: SSSD_test_ECC_cert_x509_%.pem
	$(OPENSSL) x509 -in $< -pubkey -noout > $@

SSSD_test_ECC_cert_pubsshkey_%.pub: SSSD_test_ECC_cert_pubkey_%.pem
	$(SSH_KEYGEN) -i -m PKCS8 -f $< > $@

SSSD_test_ECC_cert_x509_%.h: SSSD_test_ECC_cert_x509_%.pem
	@echo "#define SSSD_TEST_ECC_CERT_$* \""$(shell cat $< |openssl x509 -outform der | base64 -w 0)"\"" > $@

SSSD_test_ECC_cert_pubsshkey_%.h: SSSD_test_ECC_cert_pubsshkey_%.pub
	@echo "#define SSSD_TEST_ECC_CERT_SSH_KEY_$* \""$(shell cut -d' ' -f2 $<)"\"" > $@

SSSD_test_ECC_crl.pem: $(openssl_ecc_ca_key) SSSD_test_ECC_CA.pem
	$(OPENSSL) ca -gencrl -out $@ -keyfile $(openssl_ecc_ca_key) -config $(openssl_ecc_ca_config) -crldays 99999


softhsm2_ecc_one: softhsm2_ecc_one.conf
	mkdir $@
	SOFTHSM2_CONF=./$< $(SOFTHSM2_UTIL) --init-token  --label "SSSD Test ECC Token" --pin 123456 --so-pin 123456 --free
	GNUTLS_PIN=123456 SOFTHSM2_CONF=./$< $(P11TOOL) --provider=$(SOFTHSM2_PATH) --write --no-mark-private --load-certificate=SSSD_test_ECC_cert_x509_0001.pem --login  --label 'SSSD test ECC cert 0001' --id '190E513C9A3DFAACDE5D2D0592F0FDFF559C10CB'
	GNUTLS_PIN=123456 SOFTHSM2_CONF=./$< $(P11TOOL) --provider=$(SOFTHSM2_PATH) --write --load-privkey=$(srcdir)/SSSD_test_ECC_cert_key_0001.pem --login  --label 'SSSD test ECC cert 0001' --id '190E513C9A3DFAACDE5D2D0592F0FDFF559C10CB'

softhsm2_ecc_one.conf:
	@echo "directories.tokendir = "$(abs_top_builddir)"/src/tests/test_ECC_CA/softhsm2_ecc_one" > $@
	@echo "objectstore.backend = file" >> $@
	@echo "slots.removable = true" >> $@

CLEANFILES = \
    index.txt  index.txt.attr \
    index.txt.attr.old  index.txt.old \
    serial  serial.old  \
    SSSD_test_ECC_CA.pem SSSD_test_ECC_crl.pem $(pwdfile) \
    $(certs) $(certs_h) $(pubkeys) $(pubkeys_h) $(pkcs12) \
    softhsm2_*.conf \
    $(NULL)

clean-local:
	rm -rf newcerts
	rm -rf softhsm*

serial: clean
	touch index.txt
	touch index.txt.attr
	mkdir newcerts
	echo -n 01 > serial
