# Design — oh-my-nixos framework scaffold

> 配套阅读 `prd.md` 的 D1–D9 决策。本文档定义架构、边界、目录结构、option 契约、flake 集成点与取舍。

## 1. 架构总览

仓库 = **可 fork 的 opinionated NixOS flake** + **轻量生成器脚本**。整个机器配置由 *module option* 驱动；host/home 的"个性"全部是 option 的取值，不存在第二套脚本式配置机制。

```
oh-my-nixos/
├── flake.nix                  # flakelight 入口，自动加载 nix/ 与 hosts/homes
├── nix/                       # 框架代码（被 flakelight 自动 import）
│   ├── lib/                   # 纯函数辅助（mkHost/mkHome/option 工厂）
│   ├── modules/               # NixOS modules（系统层）
│   │   ├── core/              # nix settings/boot/locale/ssh/network/zram/firmware
│   │   ├── security/         # lanzaboote(opt-in) / LUKS(opt-in) / agenix 接线
│   │   ├── persistence/      # impermanence 系统层（/persist 白名单）
│   │   ├── disk/             # disko 预设布局工厂（btrfs/ext4/tmpfs-root 变体）
│   │   └── presets/          # host 预设：laptop / desktop / server
│   ├── home/                  # home-manager modules（用户层）
│   │   ├── core/             # shell/git/cli/编辑器/主题
│   │   ├── persistence/      # impermanence home 三档（ephemeral/persist-dots/persistent）
│   │   ├── graphic/          # GNOME 默认 + KDE/Hyprland/Sway opt-in切换
│   │   ├── terminal/         # 终端工具链
│   │   └── presets/          # home 预设：graphic / terminal
│   └── presets/               # option 注册：把 host/home preset enum 注册成 module option
├── hosts/                     # 每台机器一个目录
│   └── example/             # 随仓自带 laptop example
│       ├── host.nix          # 该 host 的 option 取值（preset/disk/网络/用户列表）
│       ├── hardware.nix      # 该 host 硬件（hardware-configuration 或生成器产出）
│       └── disk.nix          # 该 host 的 disko 布局（基于 presets/disk 工厂）
├── homes/                     # 每个用户一个 home module
│   └── example/
│       └── home.nix          # 该 user 的 option 取值 + override
├── secrets/                   # agenix：secrets.nix 公钥表 + *.age（默认仅占位）
├── modules/                   # 用户自定义 module 收集点（空目录 + README）
└── tools/
    └── generate/              # 轻量生成器（bash 或 nix run #generate）
        └── generate.sh
```

## 2. flake 集成点（flakelight）

`flake.nix` 不手写 `outputs`，靠 `flakelight ./. { inherit inputs; }` 自动加载 `nix/`。在 `nix/flake-module.nix`（flakelight module）内：

- 用 `perSystem` 暴露生成器为 `apps.x86_64-linux.generate` / `packages.generate`，亦支持 `nix run .#generate`。
- 用 `nixosConfigurations` 在 flakelight 的 `nixosConfigurations` 字段里：遍历 `hosts/*/host.nix`，对每个用 `mkHost` 工厂构造一个 `nixosConfiguration`（含 disko/impermanence/home-manager 引用）。
- `homeConfigurations` 不单独构造（home 通过 host 的 home-manager module 集成，避免两套 activation）；但 `homes/<user>/home.nix` 结构独立，可被多个 host 引用。

## 3. option 契约（接口面）

> 所有选项默认值集中在 `nix/presets/` 的等重要点；host/home 只填取值，不写实现逻辑。

### 3.1 host 层（`oh-my-nixos.host.*`，`nix/modules/presets/host.nix` 注册）

| option | 类型 | 默认 | 说明 |
|---|---|---|---|
| `host.name` | str | required | 机器名，等价 `networking.hostName` |
| `host.preset` | enum [laptop desktop server] | required | 主机预设 |
| `host.purpose` | str | "" | 用户备注（人类可读） |
| `host.disko` | submodule 或 null | per-preset | 由 `disk.nix` 给出，调 `disk.presets.<name>` 工厂 |
| `host.luks.enable` | bool | false | LUKS 全盘加密（D6 opt-in） |
| `host.lanzaboote.enable` | bool | false | secure boot（D6 opt-in） |
| `host.persistence.system` | bool | true | 系统层走 impermanence（true = tmpfs 根 + /persist 白名单） |
| `host.network.backend` | enum [networkmanager networkd] | per-preset | D9：graphic→nm，server→networkd |
| `host.ssh.enable` | bool | true | openssh server |
| `host.ssh.passwordAuth` | bool | false | 默认仅密钥；server 首装时可由生成器临时置 true |
| `host.arch` | enum [x86_64 aarch64] | x86_64 | D5 |
| `host.homes` | listOf str | [] | 本机启用的 user 列表（引用 `homes/<user>/home.nix`） |

### 3.2 home 层（`oh-my-nixos.home.*`，`nix/home/presets/home.nix` 注册）

| option | 类型 | 默认 | 说明 |
|---|---|---|---|
| `home.user` | str | required | 用户名 / 家目录名 |
| `home.preset` | enum [graphic terminal] | required | home 预设 |
| `home.persistence` | enum [ephemeral persist-dots persistent] | ephemeral | D4 三档 |
| `home.desktop` | enum [gnome kde hyprland sway] | gnome | D8 |
| `home.shell` | enum [fish zsh bash] | fish | D9 |
| `home.stylixScheme` | str | per-preset | stylix base16 scheme 名（见 §6） |

### 3.3 预设机制

`host.preset` 不是 "目录模板"，而是**枚举开关**：`nix/modules/presets/<preset>.nix` 里 `config = mkIf (cfg.preset == "laptop") { ... }`。这样枚举与实现同源、生成器只需填 enum、schema 稳定难走样（D1 的核心约束）。

## 4. 关键集成点实现要点

### 4.1 tmpfs 根 + impermanence（D3 B 方案）

- `host.persistence.system == true` 时：disko 把 `/` 设为 tmpfs（mount 选项），`/nix` 与 `/persist` 为持久分区（默认 ext4，可选 btrfs）；`impermanence.nix` 用 `environment.persistence."/persist"` 绑定系统层白名单（`/etc/machine-id /var/lib/nixos /var/lib/systemd/random-seed /etc/ssh/ssh_host_*_key` 等）。
- `home.persistence == ephemeral`：`/home` tmpfs，impermanence 把白名单 bind 到 `/persist/home/<user>/<path>`；预置白名单见 §5。
- `home.persistence == persistent`：`/home` 落持久分区、不进 impermanence。

### 4.2 disko 布局工厂

`nix/modules/disk/` 暴露 `mkDisk { format, luks?, impermanence? }`：
- `format = "ext4"`（默认）/ `"btrfs"`；
- `luks.enable` → 在分区层套 LUKS；
- `impermanence = true` → 生成 tmpfs 根 + /nix + /persist 三件套，否则普通可变根 + / + /boot/efi。
- host 的 `disk.nix` 调工厂填磁盘设备名（由生成器或 `nixos-generate-config` 填写）。

### 4.3 home-manager 集成（D2 务实变体）

- `mkHost` 内 `home-manager.nixosModules.home-manager` + `home-manager.users.<user> = import ../homes/<user>/home.nix`。
- `homes/<user>/home.nix` 是 module（不含 `home.username`，那是 host 在引用时填 host 实际用户名？为简单，`home.user` 即为登录名；同 user 跨 host 自然复用）。
- host 可在该 import 后再追加 `home-manager.users.<user>.<...>` override（host 专属图形应用等）。

### 4.4 agenix（D7）

- `nix/modules/security/agenix.nix`：`age.identity.identityPaths = [ /etc/ssh/ssh_host_ed25519_key ]`；`age.secrets` 由 host `host.secrets` list 驱动（默认空）。
- `secrets/secrets.nix`：`age` 公钥表，由生成器把每 host 的 `ssh-keyscan` 回填（首次 build 前），文档示范 `agenix -e wifi.age`。
- 随 example 一个占位 `secrets/example-wifi.age`（值为空字符串）+ `example/host.nix` 不实际引用它，仅证接线编译通过。

### 4.5 stylix（D9 主题）

- `nix/home/core/stylix.nix`：启用 `stylix.autoEnable = true`、按 `home.stylixScheme` 设 `base16Scheme`。
- 默认 scheme：`graphic` → `tokyonight`，`terminal` → `tokyonight-night`（统一基调，桌/端分别点亮）；option 可改。
- 字体：`stylix.fonts.sansSerif = ["Noto Sans CJK SC"]`、`monospace = ["JetBrainsMono Nerd Font"]`。

## 5. 预置 impermanence home 白名单（ephemeral 模式）

```
~/.ssh               ~/.config/git       ~/.config/fish       ~/.config/nix
~/.local/share/fish  ~/.config/starship.toml
~/.gnupg             ~/.cache/BraveSoftware  (浏览器 profile)
~/.config/Microsoft/vscode  ~/.vscode  ~/.vscode-extensions
~/.config/JetBrains  ~/.local/share/JetBrains
~/.zoxide            ~/.local/share/direnv
~/.config/fcitx5     ~/.local/state/fcitx5
~/.local/state/nix   ~/.cache/nix
```
> 列在 `nix/home/persistence/default-whitelist.nix`，home 可加项；维护此清单质量 = 旗舰体验关键（D4 代价）。

## 6. 安全默认（D6 opt-in）

- LUKS / lanzaboote / firewall-on-w/ssh-password-off；opt-in 模式下由 `host.luks.enable` `host.lanzaboote.enable` 开。
- firewall：默认 `networking.firewall.enable = true`，server 仅开 ssh（22）。
- openssh：`permitRootLogin = "prohibit-password"`，`passwordAuth = false`（option 可改）。

## 7. nixpkgs-stable 使用方式

`nix/lib/overlay.nix` 把 `nixpkgs-stable` 注入为 `pkgs.stable.<pkg>` 顶层 attr（通过 flakelight 的 `overlays` / nixos module `nixpkgs.overlays`）。host 可 `pkgs.stable.<name>` 单包 pin；默认不强制使用。

## 8. 生成器（`tools/generate`）

交互问句顺序：`host.name → host.preset(laptop/desktop/server) → host.homes[user] → home[user].preset(graphic/terminal) → home[user].persistence(ephemeral/…) → 磁盘设备路径 → 是否启用 LUKS / lanzaboote（默认否）`。输出：克隆 `hosts/example/` 到 `hosts/<name>/`，改 `host.nix` 字段，调 `disk` 工厂填 `disk.nix`，必要时 mkdir `homes/<user>/`。纯 bash + coreutils + nix（评估 enum 校验），不引外部依赖。

## 9. 兼容性 / 回滚 / 运维

- 不迁移既有系统：greenfield，无旧数据兼容负担。
- 回滚单位：git commit —— 任何变更可在主机 `nixos-rebuild boot` 后回滚上一代；impermanence tmpfs 根下"根"无 rollback，靠 git + generation 切换。disko 一旦写入磁盘不可逆（文档警告）。
- 运维注：换机器需重生成 ssh host key → 重加密所有 agenix secret（D7 代价，写进 README 警告块）。

## 10. 主要取舍小结

| 维度 | 取舍 | 代价 |
|---|---|---|
| 生成器 + option 双轨（D1） | 维护生成器与 option schema 同步 | 中，靠 enum 控制范围 |
| home 跨 host 复用 + host override（D2） | 需接线约定 | 低 |
| tmpfs 根（D3） | /nix 必须独立持久分区 | 新手首次需理解，文档补 |
| home 持久化三档（D4） | 旗舰 ephemeral 需高质量白名单 | 持续维护清单 |
| 安全 opt-in（D6） | 卖点不在默认 | 文档作为升级路径推销 |
| GNOME 默认（D8） | KDE/Hyprland 体验需作为 opt-in 验证 | 中等 |

## 11. 待落地后再决的小项

- stylix 默认 scheme 替换是否要 per-host 随机；暂统一 `tokyonight`。
- server 是否默认开 `podman`（暂 opt-in）。
- aarch64 实验性支持的 CI 关卡（暂不进 CI）。