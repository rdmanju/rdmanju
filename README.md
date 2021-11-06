Skip to content
Search or jump to…
Pull requests
Issues
Marketplace
Explore
 
@rdmanju 
SpectralOps
/
keyscope
Public
13
310105
Code
Issues
Pull requests
Actions
Projects
Wiki
Security
Insights
keyscope/.github/workflows/release.yml
@jondot
jondot open sourcing keyscope 😄
Latest commit 9da343f on Oct 1
 History
 1 contributor
218 lines (186 sloc)  6.08 KB
   
name: Release
on:
  # schedule:
  # - cron: '0 0 * * *' # midnight UTC

  push:
    tags:
    - 'v[0-9]+.[0-9]+.[0-9]+'
    ## - release

env:
  BIN_NAME: keyscope
  PROJECT_NAME: keyscope
  REPO_NAME: spectralops/keyscope
  BREW_TAP: spectralops/homebrew-tap

jobs:
  dist:
    name: Dist
    runs-on: ${{ matrix.os }}
    strategy:
      fail-fast: false # don't fail other jobs if one fails
      matrix:
        build: [x86_64-linux, x86_64-macos, x86_64-windows] #, x86_64-win-gnu, win32-msvc aarch64-linux, 
        include:
        - build: x86_64-linux
          os: ubuntu-20.04
          rust: stable
          target: x86_64-unknown-linux-gnu
          cross: false

        # - build: aarch64-linux # this fails on openssl
        #   os: ubuntu-20.04
        #   rust: stable
        #   target: aarch64-unknown-linux-gnu
        #   cross: true

        - build: x86_64-macos
          os: macos-latest
          rust: stable
          target: x86_64-apple-darwin
          cross: false
        - build: x86_64-windows
          os: windows-2019
          rust: stable
          target: x86_64-pc-windows-msvc
          cross: false
        # - build: aarch64-macos
        #   os: macos-latest
        #   rust: stable
        #   target: aarch64-apple-darwin
        # - build: x86_64-win-gnu
        #   os: windows-2019
        #   rust: stable-x86_64-gnu
        #   target: x86_64-pc-windows-gnu
        # - build: win32-msvc
        #   os: windows-2019
        #   rust: stable
        #   target: i686-pc-windows-msvc

    steps:
      - name: Checkout sources
        uses: actions/checkout@v2
        with:
          submodules: true

      - name: Install ${{ matrix.rust }} toolchain
        uses: actions-rs/toolchain@v1
        with:
          profile: minimal
          toolchain: ${{ matrix.rust }}
          target: ${{ matrix.target }}
          override: true

      - name: Install OpenSSL
        if: matrix.build == 'aarch64-linux'
        run: sudo apt-get install libssl-dev pkg-config

      - name: Run cargo test
        uses: actions-rs/cargo@v1
        with:
          use-cross: ${{ matrix.cross }}
          command: test
          args: --release --locked --target ${{ matrix.target }}

      - name: Build release binary
        uses: actions-rs/cargo@v1
        with:
          use-cross: ${{ matrix.cross }}
          command: build
          args: --release --locked --target ${{ matrix.target }}

      - name: Strip release binary (linux and macos)
        if: matrix.build == 'x86_64-linux' || matrix.build == 'x86_64-macos'
        run: strip "target/${{ matrix.target }}/release/$BIN_NAME"

      - name: Strip release binary (arm)
        if: matrix.build == 'aarch64-linux'
        run: |
          docker run --rm -v \
            "$PWD/target:/target:Z" \
            rustembedded/cross:${{ matrix.target }} \
            aarch64-linux-gnu-strip \
            /target/${{ matrix.target }}/release/$BIN_NAME
      - name: Build archive
        shell: bash
        run: |
          mkdir dist
          if [ "${{ matrix.os }}" = "windows-2019" ]; then
            cp "target/${{ matrix.target }}/release/$BIN_NAME.exe" "dist/"
          else
            cp "target/${{ matrix.target }}/release/$BIN_NAME" "dist/"
          fi
      - uses: actions/upload-artifact@v2.2.4
        with:
          name: bins-${{ matrix.build }}
          path: dist

  publish:
    name: Publish
    needs: [dist]
    runs-on: ubuntu-latest
    steps:
      - name: Checkout sources
        uses: actions/checkout@v2
        with:
          submodules: false

      - uses: actions/download-artifact@v2
        # with:
        #   path: dist
      # - run: ls -al ./dist
      - run: ls -al bins-*

      - name: Calculate tag name
        run: |
          name=dev
          if [[ $GITHUB_REF == refs/tags/v* ]]; then
            name=${GITHUB_REF:10}
          fi
          echo ::set-output name=val::$name
          echo TAG=$name >> $GITHUB_ENV
        id: tagname

      - name: Build archive
        shell: bash
        run: |
          set -ex
          rm -rf tmp
          mkdir tmp
          mkdir dist
          for dir in bins-* ; do
              platform=${dir#"bins-"}
              if [[ $platform =~ "windows" ]]; then
                  exe=".exe"
              fi
              pkgname=$PROJECT_NAME-$TAG-$platform
              mkdir tmp/$pkgname
              # cp LICENSE README.md tmp/$pkgname
              mv bins-$platform/$BIN_NAME$exe tmp/$pkgname
              chmod +x tmp/$pkgname/$BIN_NAME$exe
              if [ "$exe" = "" ]; then
                  tar cJf dist/$pkgname.tar.xz -C tmp $pkgname
              else
                  (cd tmp && 7z a -r ../dist/$pkgname.zip $pkgname)
              fi
          done
      - name: Upload binaries to release
        uses: svenstaro/upload-release-action@v2
        with:
          repo_token: ${{ secrets.GITHUB_TOKEN }}
          file: dist/*
          file_glob: true
          tag: ${{ steps.tagname.outputs.val }}
          overwrite: true


      # update homebrew

      - name: Extract version
        id: extract-version
        run: |
          printf "::set-output name=%s::%s\n" tag-name "${GITHUB_REF#refs/tags/}"
      - uses: mislav/bump-homebrew-formula-action@v1
        with:
          formula-path: ${{env.PROJECT_NAME}}.rb
          homebrew-tap: ${{ env.BREW_TAP }}
          download-url: "https://github.com/${{ env.REPO_NAME }}/releases/download/${{ steps.extract-version.outputs.tag-name }}/${{env.PROJECT_NAME}}-${{ steps.extract-version.outputs.tag-name }}-x86_64-macos.tar.xz"
          commit-message: updating formula for ${{ env.PROJECT_NAME }}
        env:
          COMMITTER_TOKEN: ${{ secrets.COMMITTER_TOKEN }}




        #
        # you can use this initial file in your homebrew-tap if you don't have an initial formula:
        # <projectname>.rb
        #
        # class <Projectname capitalized> < Formula
        #   desc "A test formula"
        #   homepage "http://www.example.com"
        #   url "-----"
        #   version "-----"
        #   sha256 "-----"

        #   def install
        #     bin.install "<bin-name>"
        #   end
        # end

© 2021 GitHub, Inc.
Terms
Privacy
Security
Status
Docs
Contact GitHub
Pricing
API
Training
Blog
About
Loading complete
