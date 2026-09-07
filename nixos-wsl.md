# NixOS - WSL2

download latest nixos.wsl from
[https://github.com/nix-community/NixOS-WSL/releases/latest]


[install]
```
wsl --install --from-file nixos.wsl

sudo nix-channel --update

sudo nixos-rebuild switch
```

[login] 
```
wsl -d NixOS
```

[Post-Install]

[set password]

```
passwd
# or
sudo passwd nixos
```

[update]
```
sudo nix-channel --update

sudo nixos-rebuild switch

sudo nixos-rebuild switch --upgrade

```

[change default useranem from nixos]

sudo nano /etc/nixos/configuration.nix

change the nixos to avinash 
```
# wsl.defaultUser = "nixos"
wsl.defaultUser = "avinash"
```

ctrl O -- to save
ctrl X -- to exit

```
sudo nixos-rebuild boot  -- dont use nixos-rebuild switch here
```
[set password - avinash]

```
sudo passwd avinash
```



sudo nano /etc/nixos/configuration.nix

[Add applications in the config]
```

# /etc/nixos/configuration.nix
environment.systemPackages = with pkgs; [
  vim
  wget
  git
  uv
];
```

ctrl O -- to save
ctrl X -- to exit

sudo nixos-rebuild switch   

[Now vim is available]

sudo vi /etc/nixos/configuration.nix

#### sample configuration.nix

```
{ config, lib, pkgs, ... }:

{
  imports = [
    # include NixOS-WSL modules
    <nixos-wsl/modules>
  ];

  wsl.enable = true;
  wsl.defaultUser = "avinash";

  environment.systemPackages = with pkgs; [
    vim
    wget
    git
    uv
    cmatrix
    fastfetch
  ];

  # This value determines the NixOS release from which the default
  # settings for stateful data, like file locations and database versions
  # on your system were taken. It's perfectly fine and recommended to leave
  # this value at the release version of the first install of this system.
  # Before changing this value read the documentation for this option
  # (e.g. man configuration.nix or on https://nixos.org/nixos/options.html).
  system.stateVersion = "26.05"; # Did you read the comment?

  nix.settings.experimental-features = [ "nix-command" "flakes" ];

}

```




add after the packages stuff

```
programs.nix-ld.enable = true;
```


sudo nixos-rebuild switch  


#### add btop

```
nix-shell -p btop-cuda
nix-shell -p htop

sudo nixos-rebuild switch

```






[installation docs]
[https://nix-community.github.io/NixOS-WSL/install.html]






[rebuild CMD]
```
flake.nix to rebuild
```


[enable flatpak]
in the config file

```
services.flatpak.enable = true;
```


#### DEV Setup - python
[create a file in the project folder - shell.nix]


```
{ pkgs ? import <nixpkgs> {} }:
pkgs.mkShell {
  packages = [
    (pkgs.python3.withPackages (python-pkgs: [
      python-pkgs.pandas
      python-pkgs.requests
      python-pkgs.polars
      python-pkgs.duckdb
      python-pkgs.pyarrow
      python-pkgs.datafusion
    ]))
  ];
}

```

[to activate - python]

```
nix-shell
```


#### DEV Setup - rust
[create a file in the project folder - flake.nix]


```
{
  description = "Rust development environment";

  inputs = {
    nixpkgs.url = "github:Nixos/nixpkgs/nixos-unstable";
  };

  outputs = { self, nixpkgs }:
    let
      system = "x86_64-linux"; # Adjust to your architecture if different
      pkgs = import nixpkgs { inherit system; };
    in {
      devShells.${system}.default = pkgs.mkShell {
        buildInputs = with pkgs; [
          cargo
          rustc
          rust-analyzer
          clippy
          rustfmt
          pkg-config
          gcc
        ];

        RUST_SRC_PATH = pkgs.rustPlatform.rustLibSrc;
      };
    };
}

```

[to activate - rust]

```
nix develop
```


#### DEV Setup - bun
[create a file in the project folder - flake.nix]


```
{
  description = "Bun development environment";

  inputs = {
    nixpkgs.url = "github:Nixos/nixpkgs/nixos-unstable";
  };

  outputs = { self, nixpkgs }:
    let
      system = "x86_64-linux"; # Adjust if you are on aarch64-linux, etc.
      pkgs = import nixpkgs { inherit system; };
    in {
      devShells.${system}.default = pkgs.mkShell {
        buildInputs = with pkgs; [
          bun
        ];

        shellHook = ''
          echo "Welcome to your Bun development environment!"
          bun --version
        '';
      };
    };
}

```

[to activate - rust]

```
nix develop
```
