#
# This makefile's targets rebuild various container images that can be used
# for development and testing in the CoCo project.
# They also are intended to serve as an up-to-date reference for creating
# new images.
#
# Note: The targets push to ghcr, which requires proper credentials and
# `docker login`.
#

.PHONY: unsig \
cosign-sig \
simple-sig \
enc-unsig \
enc-cosign-sig \
test-container-unencrypted \
test-container-encrypted \
busybox

SHELL=/bin/bash

PLATFORMS := linux/amd64 linux/s390x

define build_and_push_image
	@for platform in $(PLATFORMS); do \
		image_name="$(1)"; \
		if [ "$$platform" = "linux/s390x" ]; then \
			image_name="$(1)-s390x"; \
		fi; \
		echo "Building for $$platform with image name $$image_name..."; \
		docker buildx build \
		  -t $$image_name \
		  -f $(2) \
		  --platform $$platform --provenance=false --load $(3); \
		docker push $$image_name; \
	done
endef

# FIXME need to choose sane package URLs/names/tags
COCO_PKG=bbolroc/test-container
COCO_PKG_IMGRS=bbolroc/test-container-image-rs



all: \
	unsig \
	cosign-sig \
	simple-sig \
	enc-unsig \
	enc-cosign-sig \
	test-container-unencrypted \
	test-container-encrypted \
	busybox



unsig:
	$(call build_and_push_image,ghcr.io/$(COCO_PKG):unsig,dockerfiles/alpine-with-sshd/Dockerfile,.)


cosign-sig:
	$(call build_and_push_image,ghcr.io/$(COCO_PKG):cosign-sig,dockerfiles/alpine-with-sshd/Dockerfile,.)
	# FIXME Replace expect script with something better
	${CURDIR}/scripts/make-cosign-sig.exp $(COCO_PKG) cosign-sig


# NOTE: This depends on a gpg key owned by git@runner.com.
# That is, before issuing this make target, have to do something like:
#   $ gpg --batch --import ./keys/sign/github-runner.keys
simple-sig:
	skopeo \
	  copy \
	  --debug \
	  --insecure-policy \
	  --sign-by git@runner.com \
	  --sign-passphrase-file $(shell pwd)/keys/sign/git-runner-password.txt \
	  docker-daemon:ghcr.io/$(COCO_PKG):unsig \
	  docker://ghcr.io/$(COCO_PKG):simple-sig


# NOTE: This requires coco-keyprovider running from guest-components...
# That is, before issuing this make target, have to do something like:
#   $ cd guest-components/attestation-agent/coco_keyprovider
#   $ RUST_LOG=coco_keyprovider cargo run --release -- --socket 127.0.0.1:50000
enc-unsig: unsig
	OCICRYPT_KEYPROVIDER_CONFIG="$(shell pwd)/configs/ocicrypt.conf" \
	skopeo copy \
	  --insecure-policy \
	  --encryption-key provider:attestation-agent:keypath=$(shell pwd)/keys/encrypt/key1::keyid=kbs:///default/key/key_id1::algorithm=A256GCM \
	  docker-daemon:ghcr.io/$(COCO_PKG):unsig \
	  docker://ghcr.io/$(COCO_PKG):enc-unsig


# NOTE: see enc-unsig about coco-keyprovider
# NOTE: see cosign-sig about replacing expect script
enc-cosign-sig: cosign-sig
	OCICRYPT_KEYPROVIDER_CONFIG="$(shell pwd)/configs/ocicrypt.conf" \
	skopeo copy \
	  --insecure-policy \
	  --encryption-key provider:attestation-agent:keypath=$(shell pwd)/keys/encrypt/key1::keyid=kbs:///default/key/key_id1::algorithm=A256GCM \
	  docker-daemon:ghcr.io/$(COCO_PKG):cosign-sig \
	  docker://ghcr.io/$(COCO_PKG):enc-cosign-sig
	./scripts/make-cosign-sig.exp $(COCO_PKG) enc-cosign-sig


test-container-unencrypted:
	$(call build_and_push_image,ghcr.io/$(COCO_PKG):unencrypted,dockerfiles/alpine-with-sshd/Dockerfile,.)


# NOTE: see enc-unsig about coco-keyprovider
test-container-encrypted: test-container-unencrypted
	OCICRYPT_KEYPROVIDER_CONFIG="$(shell pwd)/configs/ocicrypt.conf" \
	skopeo copy \
	  --insecure-policy \
	  --encryption-key provider:attestation-agent:keypath=$(shell pwd)/keys/encrypt/key1::keyid=kbs:///default/key/key_id1::algorithm=A256GCM \
	  docker-daemon:ghcr.io/$(COCO_PKG):unencrypted \
	  docker://ghcr.io/$(COCO_PKG):encrypted


busybox:
	$(call build_and_push_image,ghcr.io/$(COCO_PKG_IMGRS):busybox,dockerfiles/busybox/Dockerfile,dockerfiles/busybox)
