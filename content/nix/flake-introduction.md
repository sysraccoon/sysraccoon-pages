---
date: "2025-07-30T20:29:29+03:00"
draft: true
title: "Введение в Nix Flakes"
weight: 90
tags: draft
---

Эта статья представляет из себя пошаговое введение в работу с Nix Flakes. Начиная с теоретического объяснения что это и как работает, заканчивая практической частью разбитой на две части: конфигурацию системы и разработку с использованием Flake.

## Что такое Flake?

Nix Flake - это формат описания Nix-проекта, который включает:

- `flake.nix` - описывающий зависимости и результаты сборки на языке Nix
- `flake.lock` - автоматически генерируемый файл содержащий точные версии всех зависимостей

Команды вроде `nix build`, `nix develop`, `nixos-rebuild switch` умеют работать с этим форматом получая из него нужные данные.

{{< callout type="warning" >}}
  Nix Flakes являются экспериментальной функцией и в будущем могут произойти серьёзные изменения. Данный материал актуален на июль 2025 (nixos 25.05, nix 2.28.4).

  Для работы Nix Flakes необходимо добавить следующие параметры в `/etc/nixos/configuration.nix` если используется NixOS:
  ```nix {filename="/etc/nixos/configuration.nix"}
  nix.settings.experimental-features = [ "nix-command" "flakes" ];
  ```

  Или в `/etc/nix/nix.conf` если вы используете Nix на **любом другом дистрибутиве**:
  ```ini {filename="/etc/nix/nix.conf"}
  experimental-features = nix-command flakes
  ```
{{< /callout >}}

### Про flake.nix

Чтобы начать использовать данный формат достаточно в корне проекта создать файл с именем `flake.nix`, в котором есть `inputs` и `outputs`:

```nix {filename="flake.nix"}
{
  inputs = {
    # ...
  };

  outputs = { ... }: {
    # ...
  };
}
```

В `inputs` описываются зависимости которые нужны для сборки данного проекта:
```nix {filename="flake.nix",hl_lines=[3]}
{
  inputs = {
    nixpkgs.url = "github:nixos/nixpkgs/nixos-25.05";
  };

  outputs = { ... }: {
    # ...
  };
}
```

В данном случае мы пытаемся получить доступ к содержимому nixpkgs в ветке `nixos-25.05` ([github](https://github.com/NixOS/nixpkgs/tree/nixos-25.05)). При желании можно указать сразу несколько разных версий nixpkgs:

```nix {filename="flake.nix",hl_lines=[4]}
{
  inputs = {
    nixpkgs.url = "github:nixos/nixpkgs/nixos-25.05";
    nixpkgs-unstable.url = "github:nixos/nixpkgs/nixos-unstable";
  };

  outputs = { ... }: {
    # ...
  };
}
```

`outputs` это функция принимающая загруженные и распакованные зависимости из `inputs` и возвращающая результат работы (пакеты, окружения, конфиги и т.д):

```nix {filename="flake.nix"}
{
  inputs = {
    nixpkgs.url = "github:nixos/nixpkgs/nixos-25.05";
    nixpkgs-unstable.url = "github:nixos/nixpkgs/nixos-unstable";
  };

  outputs = { nixpkgs, nixpkgs-unstable, ... }: {
    # Через packages определяются пакеты
    packages.x86_64-linux.default = /*...*/;
    # Через devShells преднастроенные dev-окружения
    devShells.x86_64-linux.default = /*...*/;
    # Через nixosConfigurations системные конфигурации для NixOS
    nixosConfigurations.hostname = /*...*/;
  };
}
```

Разными результатами пользуются разные команды Nix-а. Например, при попытке вызвать `nix build` будет задействован `packages.<system>.default`, где `<system>` это архитектура системы на которой запущен Nix. При вызове `nix develop` будет создано окружение в соответствии с `devShells.<system>.default`. Можно также явно задать название необходимого объекта, который также будет искаться в соответствии с типом команды и текущей архитектурой `nix build .#default`.

Подробный список того что может находится в `outputs` можно посмотреть в [NixOS Wiki](https://nixos.wiki/wiki/Flakes#Output_schema) или под спойлером ниже.

{{<details title="Output Schema" closed="true">}}

- `<system>` используемая архитектура, например `x86_64-linux`, `aarch64-linux`, `i686-linux`, `x86_64-darwin`
- `<name>` название атрибута, например `hello`.
- `<flake>` путь к проекту где располагается `flake.nix`.
- `<store-path>` это путь в `/nix/store/...`

```nix {filename="flake.nix"}
{
  outputs = { self, ... } @ inputs: {
    # Используется в `nix flake check`
    checks."<system>"."<name>" = derivation;
    # Используется в `nix build .#<name>`
    packages."<system>"."<name>" = derivation;
    # Используется в `nix build .`
    packages."<system>".default = derivation;
    # Используется в `nix run .#<name>`
    apps."<system>"."<name>" = {
      type = "app";
      program = "<store-path>";
    };
    # Используется в `nix run . -- <args?>`
    apps."<system>".default = { type = "app"; program = "..."; };
    # Форматтеры (alejandra, nixfmt или nixpkgs-fmt)
    formatter."<system>" = derivation;
    # Используется в nixpkgs, также доступен через `nix build .#<name>`
    legacyPackages."<system>"."<name>" = derivation;
    # Оверлеи доступные для других флейков
    overlays."<name>" = final: prev: { };
    # Оверлей по умолчанию
    overlays.default = final: prev: { };
    # NixOS модули используемые другими флейками
    nixosModules."<name>" = { config, ... }: { options = {}; config = {}; };
    # Модуль по умолчанию
    nixosModules.default = { config, ... }: { options = {}; config = {}; };
    # Используется в `nixos-rebuild switch --flake .#<hostname>`
    # nixosConfigurations."<hostname>".config.system.build.toplevel должен быть деривацией
    nixosConfigurations."<hostname>" = {};
    # Используется в `nix develop .#<name>`
    devShells."<system>"."<name>" = derivation;
    # Используется в `nix develop`
    devShells."<system>".default = derivation;
    # Задачи сборки для Hydra
    hydraJobs."<attr>"."<system>" = derivation;
    # Используется в `nix flake init -t <flake>#<name>`
    templates."<name>" = {
      path = "<store-path>";
      description = "Описание шаблона";
    };
    # Используется в `nix flake init -t <flake>`
    templates.default = { path = "<store-path>"; description = ""; };
  }
}

```

Есть также дополнительные варианты используемые некоторыми другими инструментами, как пример [home-manager](https://github.com/nix-community/home-manager) может использовать конфигурацию определяемую через `homeConfigurations.<name>`

{{</details>}}

### Про flake.lock

Как упоминалось ранее, помимо `flake.nix` есть также файл `flake.lock`. Он выполняет ту же функцию что `package-lock.json` в npm (JS) или `uv.lock` в uv (Python), а именно фиксирует точные версии завимостей.

`flake.lock` создаётся автоматически при первой попытке использовать `flake.nix` через какую-либо команду (например `nix build`) и представляет собой json такого вида:

```json {filename="flake.lock"}
{
  "nodes": {
    "nixpkgs": {
      "locked": {
        "lastModified": 1753749649,
        "narHash": "sha256-+jkEZxs7bfOKfBIk430K+tK9IvXlwzqQQnppC2ZKFj4=",
        "owner": "nixos",
        "repo": "nixpkgs",
        "rev": "1f08a4df998e21f4e8be8fb6fbf61d11a1a5076a",
        "type": "github"
      },
      "original": {
        "owner": "nixos",
        "ref": "nixos-25.05",
        "repo": "nixpkgs",
        "type": "github"
      }
    },
    "nixpkgs-unstable": {
      "locked": {
        "lastModified": 1753694789,
        "narHash": "sha256-cKgvtz6fKuK1Xr5LQW/zOUiAC0oSQoA9nOISB0pJZqM=",
        "owner": "nixos",
        "repo": "nixpkgs",
        "rev": "dc9637876d0dcc8c9e5e22986b857632effeb727",
        "type": "github"
      },
      "original": {
        "owner": "nixos",
        "ref": "nixos-unstable",
        "repo": "nixpkgs",
        "type": "github"
      }
    },
    "root": {
      "inputs": {
        "nixpkgs": "nixpkgs",
        "nixpkgs-unstable": "nixpkgs-unstable"
      }
    }
  },
  "root": "root",
  "version": 7
}
```

В дальнейшем изменение в `flake.lock` происходят только если изменяются `inputs` в `flake.nix` или при явном вызове `nix flake update` что подтягивает новые версии.

Всё это позволяет увеличить вероятность что проект успешно собираемый в одной системе, соберётся в другой.

### Про Git

Команды работающие с `flake.nix` учитывают наличие `.git` в директории проекта и начинают вести себя несколько иначе чем без неё так как дают внутри `outputs` доступ только к тем директориям и файлам, что добавлены в индекс git-а. Если увидите ошибку такого вида:

```sh
error: path '/nix/store/1v04p2g4bdj0mc3z83pk9slza41i5ll6-source/configuration.nix' does not exist
```

То хоть тут и нет упоминания git, проблема скорее всего связана с ним. Достаточно добавить файл через `git add` и он будет доступен в дальнейшем внутри `flake.nix`.

Хоть ошибка и не вносит ясность в происходящее, подобный контроль позволяет избежать ситуации когда на вашей системе всё работает и собирается, а на другой из-за недостающего файла нет.

Другой важный момент, это то что `flake.lock` нужно коммитить с остальными файлами и не добавлять его в `.gitignore`, поскольку это позволит другим разработчикам (или вам же в будущем) получить такие же версии всех зависимостей.

## Конфигурация с Flake

Данная часть показывает как можно использовать `flake.nix` для работы с конфигурацией NixOS и Home Manager

### Конфигурация NixOS

Предполагаю что у вас уже есть рабочая конфигурация в `/etc/nixos` (`configuration.nix` и `hardware-configuration.nix`):

```
/etc/nixos
├── configuration.nix
└── hardware-configuration.nix
```

Их содержимое не сильно важно для создания рабочего Nix Flake, но на всякий случай ниже можно найти те что исползовались при написании данной статьи:

{{<details title="Мой конфиг NixOS" closed="true">}}

```nix {filename="/etc/nixos/configuration.nix"}
{ config, lib, pkgs, ... }:

{
  imports = [
    ./hardware-configuration.nix
  ];

  system.stateVersion = "25.05";

  boot.loader.systemd-boot.enable = true;
  boot.loader.efi.canTouchEfiVariables = true;

  networking.hostName = "demo-nixos";
  networking.networkmanager.enable = true;

  users.users.raccoon = {
    isNormalUser = true;
    extraGroups = [ "wheel" ];
  };

  environment.systemPackages = with pkgs; [
    vim
  ];

  nix.settings.experimental-features = [
    "nix-command" "flakes"
  ];

  services.openssh = {
    enable = true;
  };

  networking.firewall.allowedTCPPorts = [
    22
  ];
}
```

```nix {filename="/etc/nixos/hardware-configuration.nix"}
# Do not modify this file!  It was generated by ‘nixos-generate-config’
# and may be overwritten by future invocations.  Please make changes
# to /etc/nixos/configuration.nix instead.
{ config, lib, pkgs, modulesPath, ... }:

{
  imports =
    [ (modulesPath + "/profiles/qemu-guest.nix")
    ];

  boot.initrd.availableKernelModules = [ "ahci" "xhci_pci" "virtio_pci" "sr_mod" "virtio_blk" ];
  boot.initrd.kernelModules = [ ];
  boot.kernelModules = [ "kvm-amd" ];
  boot.extraModulePackages = [ ];

  fileSystems."/" =
    { device = "/dev/disk/by-label/NIXROOT";
      fsType = "btrfs";
    };

  fileSystems."/boot" =
    { device = "/dev/disk/by-label/NIXBOOT";
      fsType = "vfat";
      options = [ "fmask=0022" "dmask=0022" ];
    };

  swapDevices = [ ];

  # Enables DHCP on each ethernet and wireless interface. In case of scripted networking
  # (the default) this is the recommended approach. When using systemd-networkd it's
  # still possible to use this option, but it's recommended to use it in conjunction
  # with explicit per-interface declarations with `networking.interfaces.<interface>.useDHCP`.
  networking.useDHCP = lib.mkDefault true;
  # networking.interfaces.enp1s0.useDHCP = lib.mkDefault true;

  nixpkgs.hostPlatform = lib.mkDefault "x86_64-linux";
}
```

{{</details>}}

Начнём с создания `flake.nix` в котором укажем что необходимо получить nixpkgs версии `nixos-25.05`:

```nix {filename="/etc/nixos/configuration.nix",hl_lines=[3]}
{
  inputs = {
    nixpkgs.url = "github:nixos/nixpkgs/nixos-25.05";
  };

  outputs = { nixpkgs, ... }: {
    # ...
  };
}
```

Команда `sudo nixos-rebuild switch` проверяет наличие `/etc/nixos/flake.nix` и пытается найти в `outputs` значение `nixosConfiguration.<hostname>` где `<hostname>` это имя хоста для которого применяется конфиг:

```sh
$ hostname
demo-nixos # у вас оно может быть другим
```

В `outputs` добавляем соответствующее поле и на место значения подставляем вызов `lib.nixosSystem` который используя переданные параметры создаёт объект конфигурации для NixOS (подробнее можно почитать в исходниках: [nixpkgs/flake.nix](https://github.com/NixOS/nixpkgs/blob/42cf70e84aba9030223cd2548a8672c1f1887316/flake.nix#L64))

```nix {filename="/etc/nixos/configuration.nix",hl_lines=[8,9,10]}
{
  inputs = {
    nixpkgs.url = "github:nixos/nixpkgs/nixos-25.05";
  };

  outputs = { nixpkgs, ... }: {
    nixosConfigurations.demo-nixos = nixpkgs.lib.nixosSystem {
      modules = [
        ./configuration.nix
      ];
    };
  };
}
```

Пересборка системы первый раз займёт некоторое время поскольку версия `nixpkgs` определённая в системе (в `nix-channel` если быть точнее) может не совпадать с тем что определён в `flake.nix`:

```sh
$ sudo nixos-rebuild switch
```

Последующая пересборка будет использовать зависимости определённые в `flake.nix` и в `flake.lock` и не опираться на содержимое `nix-channel`. Для полного обновления системы нужно обновлять `flake.lock` с последующей пересборкой системы. Делается это следующим образом:

```sh
$ cd /etc/nixos
$ sudo nix flake update
$ sudo nixos-rebuild switch
```

Иногда может понадобиться обновить значение только одной зависимости:

```sh
$ cd /etc/nixos
$ sudo nix flake update nixpkgs
$ sudo nixos-rebuild switch
```

### Конфигурация Home Manager

> Coming Soon

## Разработка с Flake

Данная часть демонстрирует как можно использовать `flake.nix` в разработке, на примере простейшего проекта на `go`.

### Создание dev-окружения

Предположим есть простейший проект состоящий из одного файла `main.go`:

```go {filename="main.go"}
package main

import (
    "fmt"
    "os"
)

func main() {
    user := os.Getenv("USER")
    fmt.Printf("hello %s\n", user);
}
```

Для работы с ним понадобится установить `go`. Это можно сделать глобально в `configuration.nix` или для отдельного пользователя через `home-manager`, но в данном случае создадим dev-окружение что позволит определить все требуемые пакеты и переменные окружения в рамках проекта, чтобы каждый кто захочет поработать над ним мог выполнить команды:

```sh
$ git clone $REPO_URL
$ nix develop
```

И получить все необходимые зависимости тех же версий что и у нас.

Начнём с создания `flake.nix` в котором укажем что необходимо получить nixpkgs версии `nixos-25.05`:

```nix {filename="flake.nix",hl_lines=[3]}
{
  inputs = {
    nixpkgs.url = "github:nixos/nixpkgs/nixos-25.05";
  };

  outputs = { nixpkgs, ... }: {
    # ...
  };
}
```

В `outputs` импортируем содержимое `nixpkgs` указывая архитектуру с которой будем работать:

```nix {filename="flake.nix",hl_lines=[7]}
{
  inputs = {
    nixpkgs.url = "github:nixos/nixpkgs/nixos-25.05";
  };

  outputs = { nixpkgs, ... }: let
    pkgs = import nixpkgs { system = "x86_64-linux"; };
  in {
    # ...
  };
}
```

Для создания dev-окружения можно воспользоваться [pkgs.mkShell](https://nixos.org/manual/nixpkgs/stable/#sec-pkgs-mkShell):

```nix {filename="flake.nix",hl_lines=[7]}
{
  inputs = {
    nixpkgs.url = "github:nixos/nixpkgs/nixos-25.05";
  };

  outputs = { nixpkgs, ... }: let
    pkgs = import nixpkgs { system = "x86_64-linux"; };
  in {
    devShells.x86_64-linux.default = pkgs.mkShell {};
  };
}
```

Уже сейчас можно запустить `nix develop` и оказаться в bash:

```sh
$ nix develop
$ echo $SHELL
/nix/store/ih68ar79msmj0496pgld4r3vqfr7bbin-bash-5.2p37/bin/bash
$ exit # или можно использовать ctrl+d
```

Добавим `go` в создаваемое окружение (список доступных пакетов можно найти [тут](https://search.nixos.org/packages?channel=25.05&type=packages)):

```nix {filename="flake.nix",hl_lines=[10]}
{
  inputs = {
    nixpkgs.url = "github:nixos/nixpkgs/nixos-25.05";
  };

  outputs = { nixpkgs, ... }: let
    pkgs = import nixpkgs { system = "x86_64-linux"; };
  in {
    devShells.x86_64-linux.default = pkgs.mkShell {
      packages = [ pkgs.go ];
    };
  };
}
```

И попробуем ещё раз:

```sh
$ which go
go not found
$ nix develop
$ which go
/nix/store/9s1r393dnb5mygiq5f9yxy76nxpkz1gw-go-1.24.4/bin/go
```

Отлично, теперь можно запустить написанное приложение:

```sh
$ go run main.go
hello raccoon
$ exit
```

Чтобы поменять переменные окружения, достаточно прописать их в параметрах `mkShell`:

```nix {filename="flake.nix",hl_lines=[11]}
{
  inputs = {
    nixpkgs.url = "github:nixos/nixpkgs/nixos-25.05";
  };

  outputs = { nixpkgs, ... }: let
    pkgs = import nixpkgs { system = "x86_64-linux"; };
  in {
    devShells.x86_64-linux.default = pkgs.mkShell {
      packages = [ pkgs.go ];
      USER = "capybara";
    };
  };
}
```

Теперь вызвав `nix develop`, можно убедиться что в переменной `USER` находится значение `capybara`:

```sh
$ nix develop
$ echo $USER
capybara
$ go run main.go
hello capybara
```

Разобравшись с основной задачей, можно заняться рефакторингом. Вынесем архитектуру в отдельную переменную:

```nix {filename="flake.nix",hl_lines=[7,9,10,13]}
{
  inputs = {
    nixpkgs.url = "github:nixos/nixpkgs/nixos-25.05";
  };

  outputs = { nixpkgs, ... }: let
    system = "x86_64-linux";
    pkgs = import nixpkgs {
      # inherit system; эквивалентно system = system;
      inherit system;
    };
  in {
    devShells.${system}.default = pkgs.mkShell {
      packages = [ pkgs.go ];
      USER = "capybara";
    };
  };
}
```

В данном случае мы привязаны к одной платформе `x86_64-linux`, но что если нужно иметь окружения и под другие (`aarch64-linux`, `aarch64-darwin`...)? Можно конечно вручную скопировать текущий код под каждую из платформ, но лучше воспользоваться средствами которые предоставляет библиотека в nixpkgs, а именно [genAttrs](https://nixos.org/manual/nixpkgs/stable/#function-library-lib.attrsets.genAttrs). `genAttrs` принимает список строк и функцию которую нужно применить для каждого элемента из списка, на выходе возвращается объект у которого ключём выступает строка из списка, а значением результат вызова функции с этим ключём. Проще всего понять на примере:

```nix
nixpkgs.lib.genAttrs [
  "x86_64-linux"
  "aarch64-linux"
  "x86_64-darwin"
  "aarch64-darwin"
] (system: system + "_value")
=> {
  aarch64-darwin = "aarch64-darwin_value";
  aarch64-linux = "aarch64-linux_value";
  x86_64-darwin = "x86_64-darwin_value";
  x86_64-linux = "x86_64-linux_value";
}
```

С её помощью можно переписать `flake.nix` следующим образом:

```nix {filename="flake.nix"}
{
  inputs = {
    nixpkgs.url = "github:nixos/nixpkgs/nixos-25.05";
  };

  outputs = { nixpkgs, ... }: let
    supportedSystems = [ "x86_64-linux" "aarch64-linux" "x86_64-darwin" "aarch64-darwin" ];
    forAllSystems = nixpkgs.lib.genAttrs supportedSystems;
  in {
    devShells = forAllSystems (system: let
      pkgs = import nixpkgs { inherit system; };
    in {
      default = pkgs.mkShell {
        packages = [ pkgs.go ];
        USER = "capybara";
      };
    });
  };
}
```

`supportedSystems` содержит список архитектур для которых нужно создать dev-окружение. `forAllSystems` это функция созданная через `genAttrs` у которой первый аргумент (список ключей) выставлен на основе `supportedSystems`. Данная функция вызывается для создания `devShells`, подставляя вместо `system` элементы из `supportedSystems`. В результате `devShells` содержит объект такого вида:

```nix
{
  aarch64-darwin = { default = /* aarch64-darwin shell */; };
  aarch64-linux = { default = /* aarch64-linux shell */; };
  x86_64-darwin = { default = /* x64_64-darwin shell */; };
  x86_64-linux = { default = /* x86_64-linux shell */; };
}
```

Таким образом вызывая `nix develop` сможет работать на любой из перечисленных систем.

### Сборка приложения

Для демонстрации будет использоваться всё тоже приложение на go:

```go {filename="main.go"}
package main

import (
    "fmt"
    "os"
)

func main() {
    user := os.Getenv("USER")
    fmt.Printf("hello %s\n", user);
}
```

Чтобы его можно было собирать с помощью `go build` и устанавливать через `go install` добавляем рядом `go.mod`:

```gomod {filename="go.mod"}
module hello

go 1.24.4
```

Наша задача получить готовый бинарный файл `hello` используя команду `nix build`. В целях уменьшения объёма кода я буду демонстрировать новый `flake.nix`, но ничего не мешает иметь в одном файле dev-окружение и пакеты:

```nix {filename="flake.nix"}
{
  inputs = {
    nixpkgs.url = "github:nixos/nixpkgs/nixos-25.05";
  };

  outputs = { nixpkgs, ... }: let
    system = "x86_64-linux";
    pkgs = import nixpkgs { inherit system; };
  in {
    # ...
  };
}
```

Для сборки приложений на go используется функция [pkgs.buildGoModule](https://nixos.org/manual/nixpkgs/stable/#ssec-language-go):

```nix {filename="flake.nix"}
{
  inputs = {
    nixpkgs.url = "github:nixos/nixpkgs/nixos-25.05";
  };

  outputs = { nixpkgs, ... }: let
    system = "x86_64-linux";
    pkgs = import nixpkgs { inherit system; };
  in {
    packages.${system}.default = pkgs.buildGoModule {
      name = "hello";
      src = ./.;
      vendorHash = null;
    };
  };
}
```

Стоит обратить внимание на `vendorHash` выставленный в `null`. Если у вашего проекта нет дополнительных зависимостей как в данном случае, то его нужно устанавливать в `null`. Если зависимости есть то можно задать `vendorHash` в виде пустой строки `""` (или `pkgs.lib.fakeHash`) и вызвать один раз `nix build`, в ошибке будет указан какой хеш ожидается, его и нужно будет указать в `flake.nix`. Подробнее можно почитать в документации: [vendorHash](https://nixos.org/manual/nixpkgs/stable/#var-go-vendorHash).

Теперь приложение можно собрать с помощью `nix build`:

```sh
$ nix build
$ file result/bin/hello
result/bin/hello: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), statically linked, not stripped
$ result/bin/hello
hello raccoon
```

Запустить через `nix run` без создания ссылки в `result`:

```sh
$ nix run
hello raccoon
```

Установить в локальный профиль:

```sh
$ nix profile install
$ which hello
/home/raccoon/.nix-profile/bin/hello
$ hello
hello raccoon
```

И удалить его:
```sh
$ nix profile list
Name:               hello
Flake attribute:    packages.x86_64-linux.default
Original flake URL: path:/store/projects/dev/personal/hello
Locked flake URL:   path:/store/projects/dev/personal/hello?lastModified=...
Store paths:        /nix/store/7cj8cqzgja27d05522g7n5i2padl2ig5-hello
$ nix profile remove hello
$ hello
hello: command not found
```

Также как с `devShells` можно использовать `genAttrs` для того чтобы собирать приложение под разную архитектуру:

```nix {filename="flake.nix"}
{
  inputs = {
    nixpkgs.url = "github:nixos/nixpkgs/nixos-25.05";
  };

  outputs = { nixpkgs, ... }: let
    supportedSystems = [ "x86_64-linux" "aarch64-linux" "x86_64-darwin" "aarch64-darwin" ];
    forAllSystems = nixpkgs.lib.genAttrs supportedSystems;
  in {
    packages = forAllSystems (system: let
      pkgs = import nixpkgs { inherit system; };
    in {
      default = pkgs.buildGoModule {
        name = "hello";
        src = ./.;
        vendorHash = null;
      };
    });
  };
}
```

Если запустить nix build на `x86_64-linux` с явным указанием `aarch64-linux`, то получим вот такую ошибку:

```sh
$ nix build .#packages.aarch64-linux.default
error: a 'aarch64-linux' with features {} is required to build '/nix/store/dys9fm1n2qbi5518r7bm7bgfc7yixhfd-hello.drv',
but I am a 'x86_64-linux' with features {benchmark, big-parallel, kvm, nixos-test}
```

В NixOS есть возможность эмуляции для сборки под другие архитектуры (но всё также будет ограничено linux-ом, собрать приложение под MacOS так не выйдет). Редактируем `configuration.nix` добавляя в него систему которую нужно эмулировать:

```nix {filename="/etc/nixos/configuration.nix",hl_lines=[3]}
{ pkgs, lib, ... }: {
  # ...
  boot.binfmt.emulatedSystems = [ "aarch64-linux" ];
  # ...
}
```

Вызываем пересборку системы:

```sh
$ sudo nixos-rebuild switch
```

И пробуем повторно собрать приложение под ARM:

```sh
$ nix build .#packages.aarch64-linux.default
$ file result/bin/hello
result/bin/hello: ELF 64-bit LSB executable, ARM aarch64, version 1 (SYSV), statically linked, not stripped
$ result/bin/hello
hello raccoon
```

Про то как работать с другими языками в nix можно посмотреть в [документации](https://nixos.org/manual/nixpkgs/stable/#chap-language-support) или на реальных примерах в [nixpkgs](https://github.com/NixOS/nixpkgs).
