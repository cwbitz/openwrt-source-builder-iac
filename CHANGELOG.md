# Changelog

## [0.1.3](https://github.com/cwbitz/openwrt-source-builder-iac/compare/v0.1.2...v0.1.3) (2026-11-05)


### Bug Fixes

* **artifacts:** drop the previous fetch before copying a new one ([#7](https://github.com/cwbitz/openwrt-source-builder-iac/issues/7)) ([bb59bca](https://github.com/cwbitz/openwrt-source-builder-iac/commit/bb59bca9ee8b975cced471cf33d933618d3362f8))

## [0.1.2](https://github.com/cwbitz/openwrt-source-builder-iac/compare/v0.1.1...v0.1.2) (2026-11-05)


### Bug Fixes

* **build:** enable IMAGEOPT so the version options are not dropped ([#5](https://github.com/cwbitz/openwrt-source-builder-iac/issues/5)) ([a96e86b](https://github.com/cwbitz/openwrt-source-builder-iac/commit/a96e86bde1a3a7227629e6fe0960d08e51baf27e))

## [0.1.1](https://github.com/cwbitz/openwrt-source-builder-iac/compare/v0.1.0...v0.1.1) (2026-11-05)


### Features

* **build:** put the source version back in the firmware file names ([#3](https://github.com/cwbitz/openwrt-source-builder-iac/issues/3)) ([475114a](https://github.com/cwbitz/openwrt-source-builder-iac/commit/475114ad07dd74216bcca6b39cd3501b8aec7477))

## 0.1.0 (2026-11-05)


### ⚠ BREAKING CHANGES

* **schema:** `target_profile` and `source.build_image` are no longer variant keys; move the profile under `source` and drop `build_image`.

### Features

* **artifacts:** name the artifact directory after a variant digest ([dccca39](https://github.com/cwbitz/openwrt-source-builder-iac/commit/dccca39ee35d98f5ea4465615a8d0343dde22b0d))
* **build-image:** namespace the image and consume an override as-is ([984ba09](https://github.com/cwbitz/openwrt-source-builder-iac/commit/984ba09caba46e242bc9044eb53d776051cf057d))
* **build-image:** resolve the build image per variant and build local layers ([1d36fcb](https://github.com/cwbitz/openwrt-source-builder-iac/commit/1d36fcbb8add5fe97071a92a5a6765f96de0df59))
* **build-image:** ship a tracked example for machine-local layers ([f802776](https://github.com/cwbitz/openwrt-source-builder-iac/commit/f8027768a826b2a2b4e66082114c9c3303f607f8))
* **buildroot:** add the source build role ([f142e5e](https://github.com/cwbitz/openwrt-source-builder-iac/commit/f142e5e056c64c89817b142ee9e9614c2cf9c39e))
* **helpers:** add the shared vault and secret helpers ([9f9bdfe](https://github.com/cwbitz/openwrt-source-builder-iac/commit/9f9bdfe32b2bd91ee7cd8594e6814dab5cd47291))
* **inventory:** add operator configuration, examples and knob validation ([e45fedb](https://github.com/cwbitz/openwrt-source-builder-iac/commit/e45fedbe3c2105eac9c4a28d372b779a8aba8493))
* **playbooks:** add bootstrap, build, info and maintenance playbooks ([ccb02fd](https://github.com/cwbitz/openwrt-source-builder-iac/commit/ccb02fd12aef00c8a4427be58ecffb47dba0367c))
* **scripts:** add the control-node helper scripts ([ed030d2](https://github.com/cwbitz/openwrt-source-builder-iac/commit/ed030d22854add45bf2b46ed86b49eb030bb68a5))
* **sshd:** add opt-in key-only SSH hardening ([d84e469](https://github.com/cwbitz/openwrt-source-builder-iac/commit/d84e46929ed8400119697506ef2c46ecb616197b))
* **vault:** store connection secrets per machine ([6fdd77c](https://github.com/cwbitz/openwrt-source-builder-iac/commit/6fdd77c053c1557566d262e1893e500c56d4de5b))


### Bug Fixes

* **artifacts:** make the control-node artifact directory self-describing ([6045986](https://github.com/cwbitz/openwrt-source-builder-iac/commit/6045986ced5c055334cabbc10ce02262b8e67400))
* **bootstrap:** derive an unmarked admin hash with the default rounds ([5c262a7](https://github.com/cwbitz/openwrt-source-builder-iac/commit/5c262a790ec4ad0863c635ef4904f33898132d97))
* **bootstrap:** keep the probe credentials out of the change count ([56def90](https://github.com/cwbitz/openwrt-source-builder-iac/commit/56def90667ac846f9bcd24030d00c2b315d45910))
* **build-image:** add cmake, ninja and pkg-config and bump the tag ([8434d92](https://github.com/cwbitz/openwrt-source-builder-iac/commit/8434d92b3efb6a58b36fccd3dc68cfc611f54afd))
* **build-image:** cover the full host tool set and bump the tag ([355138c](https://github.com/cwbitz/openwrt-source-builder-iac/commit/355138c10bef2ffd9be9c7f2b9e4869c36abedd4))
* **build-image:** rebuild a local layer when the project image changes ([8b1654c](https://github.com/cwbitz/openwrt-source-builder-iac/commit/8b1654cba8c263140ea218a8ad9f58dcdeab3d1b))
* **build:** build the images with world ([ac7cff2](https://github.com/cwbitz/openwrt-source-builder-iac/commit/ac7cff2c7b21568da89e9cdb600d9d9aa9a1d1cb))
* **build:** enable the shared ccache through the developer options ([991035a](https://github.com/cwbitz/openwrt-source-builder-iac/commit/991035a6bbb4cd004b4a26ad70c2678dfca17ed5))
* **build:** make the defconfig step reliable and support =n options ([6413da8](https://github.com/cwbitz/openwrt-source-builder-iac/commit/6413da84c77c96f06c6ce9e8a501a5e4a665c128))
* **build:** pass OPENWRT_JOBS to the container as a string ([374b600](https://github.com/cwbitz/openwrt-source-builder-iac/commit/374b600018ebfee618f634a2489d9c274f00c3d9))
* **build:** retry world once when a parallel host tool races its dependency ([c5f3f75](https://github.com/cwbitz/openwrt-source-builder-iac/commit/c5f3f75342d89b75fbd3226fabe68ac6228c6e84))
* **build:** verify the target kernel config, not the toolchain's ([95d367a](https://github.com/cwbitz/openwrt-source-builder-iac/commit/95d367adbb67654af170806b26887e6e74992460))
* **knobs:** accept a build image reference without an explicit tag ([6790d58](https://github.com/cwbitz/openwrt-source-builder-iac/commit/6790d585e8983b1bab70f561240409280b394203))
* **preflight:** require GNU make on the build host ([839c7a9](https://github.com/cwbitz/openwrt-source-builder-iac/commit/839c7a9cbcfba76e988977e057568bf01e1cc3da))
* **preflight:** warn when reported memory throttles the job count ([d589638](https://github.com/cwbitz/openwrt-source-builder-iac/commit/d5896380e4b3f054b0598258b5b84e724c67b66b))
* **privacy:** reject tracked vault backup files ([09e6ce9](https://github.com/cwbitz/openwrt-source-builder-iac/commit/09e6ce96859bce4c7fe5c381f434f7db7a8c0f77))


### Code Refactoring

* **schema:** nest the device profile and pin one build image ([df04690](https://github.com/cwbitz/openwrt-source-builder-iac/commit/df04690840f725deeb48d27d974929cf1757215c))

## Changelog
