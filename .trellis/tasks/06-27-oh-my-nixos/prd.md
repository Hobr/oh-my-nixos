# oh-my-nixos framework scaffold

## Goal

构建名为 `oh-my-nixos` 的可 fork opinionated NixOS flake 框架，让从未接触过 Nix/NixOS 的用户能快速生成一套**可安装、功能完备**的 NixOS 配置。框架以 module option 驱动 host/home 个性，允许**多 host、多 home**，并提供 host 预设 `laptop/desktop/server`、home 预设 `graphic/terminal`、内置基础功能。flake 以 `flakelight` 组织，集成 `impermanence/disko/agenix/lanzaboote/home-manager/stylix/nix-vscode-extensions` 等 input 并实际生效。

## Background / Confirmed Facts

- 仓库 `/mnt/data/Project/oh-my-nixos` 已 git 初始化，分支 `main`、干净。
- `flake.nix` 已声明全部目标 input（nixpkgs unstable-small + unstable、flakelight、impermanence、lanzaboote、disko、home-manager、agenix、nix-vscode-extensions、stylix）且 `flake.lock` 已锁定；当前仅 `flakelight ./. { inherit inputs; }`，未导出任何 nixosConfigurations/模块。
- 仓库无任何 `hosts/homes/modules/nix/secrets/tools` 目录，纯 greenfield。
- stylix 已传递性引入 base16 等主题依赖。
- `.trellis/spec` 仅有通用 thinking guides，无 NixOS 相关约定。

## Requirements

- **R1 交付形态（D1）**：可 fork opinionated 仓库为主 + 轻量生成器。host/home 个性完全由 module option 取值表达；生成器只按交互问句落盘几个 option 值，不引入第二套配置机制。
- **R2 多 host、多 home（D2）**：`homes/<user>/` 为独立 home module（用户环境，可跨 host 复用）；`hosts/<host>/` 引用 `homes/<user>/` 并补 host 专属 override；home-manager 以 NixOS module 集成（不另起 activation）。
- **R3 状态与磁盘（D3, D4, D6）**：
  - tmpfs 根默认：`/` 为 tmpfs 重启清空，`/nix`、`/persist` 为独立持久分区。
  - impermanence 系统层走 `/persist` 白名单默认开。
  - home 持久化三档可配置 `ephemeral`（旗舰默认，预置 dot 白名单）/`persist-dots`/`persistent`。
  - LUKS 全盘加密与 lanzaboote secure boot 均 **opt-in**（默认关）。
- **R4 预设（D8）**：host 预设 `laptop/desktop/server`、home 预设 `graphic/terminal` 为 enum option；home graphic 默认 GNOME，opt-in 切 KDE/Hyprland/Sway。
- **R5 基础功能（D9）**：
  - 系统层（所有 host）：nix flakes 默认开 + 自动 gc/optimise + 二进制缓存；systemd-boot；时区 `Asia/Shanghai`、locale `zh_CN.UTF-8 + en_US.UTF-8`；openssh server（默认仅密钥登录）；fwupd + microcode + linux-firmware；zram；polkit/sudo。
  - 网络：graphic 用 networkmanager；server 用 systemd-networkd + resolved；firewall 默认开。
  - 图形（graphic）：GNOME + pipewire/wireplumber + 蓝牙 + Noto CJK/emoji/等宽字体 + fcitx5 + HiDPI；CUPS opt-in。
  - home 层（所有 user）：默认 shell fish；neovim + vscode(nix-vscode-extensions)；git + 现代 cli(zoxide/eza/bat/fzf/ripgrep/starship)；stylix 主题统一（graphic/terminal 各一套，默认 `tokyonight` 同基）。
  - 预设差异：laptop 加电源管理 + 触控板；desktop 性能向、无省电；server 无图形/音频、systemd-networkd、docker/podman opt-in。
- **R6 secrets（D7）**：默认不预置真 secret；提供 agenix module 接线 + `secrets/` 目录约定 + example 占位 secret + 文档示范加密流程；host 公钥走 ssh ed25519 派生（免额外 age key 步骤）。
- **R7 inputs 生效**：impermanence/disko/agenix/lanzaboote/home-manager/stylix/nix-vscode-extensions 不得仅停留在 input 声明，须在 module 中实际接线生效；nixpkgs-stable 以 `pkgs.stable.<pkg>` overlay 形式提供单包 pin 能力。
- **R8 flake 组织**：以 `flakelight` 自动加载 `nix/`，手写 `outputs` 不允许；生成器以 `apps.generate` / `nix run .#generate` 暴露。
- **R9 MVP 范围（D5）**：随仓提供 1 个 example host（laptop 预设）+ 1 个 example user（graphic 预设），fork 即可 `nixos-rebuild` / `disko` 验证；架构首版只保证 x86_64-linux，aarch64/linux best-effort 不进 CI 关卡。

## Acceptance Criteria

- [ ] `nix flake check` 全绿。
- [ ] `nixos-rebuild build --flake .#example` 编译通过（不需真机）。
- [ ] `nix build .#nixosConfigurations.example.config.system.build.diskoScript --dry-run` 通过（验证 disko schema）。
- [ ] example host dump 出的 option：`host.preset=laptop`、`host.homes=[example]`、`home.preset=graphic`、`home.persistence=ephemeral`、`home.desktop=gnome`，且 LUKS/lanzaboote 默认关。
- [ ] impermanence 系统层白名单（含 `/etc/ssh/ssh_host_*_key` 等）与 home 预置 dot 白名单（shell history/ssh/git/浏览器 profile/IDE 配置等）均存在并可 eval。
- [ ] 生成器交互产出 `hosts/<fake>/` 与必要 `homes/<user>/`，eval 该新生成 nixosConfiguration 成功。
- [ ] 切 `home.persistence` 至 `persist-dots` 与 `persistent` 两档均能构出 nixosConfiguration。
- [ ] 切 `home.desktop` 至 `kde/hyprland`(opt-in) 不破构建（GNOME 为默认）。
- [ ] README 含"如何 fork/生成/安装/换机器重加密 secret"四段说明 + impermanence tmpfs 根警示。

## Out of Scope

- aarch64-linux 正式 CI 支持（best-effort 实验性）。
- 真实生产 secret 内容（仅占位示范）。
- 既有系统的迁移工具（greenfield, 无旧数据）。
- 服务器裸金属远程 LUKS 解锁自动化（首版由文档指导，不内建）。
- 多 NixOS channel 切换工具（仅 unstable-small + stable overlay 两种）。

## Open Questions (blocking planning)

暂无残留阻塞性问题（stylix 默认 scheme、server 是否默认 podman、aarch64 CI 关卡已记入 design §11 待落地后再决）。