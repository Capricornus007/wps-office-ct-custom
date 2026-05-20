# Maintainer: Capricornus007 <sihaogang at gmail dot com>
pkgbase=wps-office-ct-custom
pkgname=('wps-office-ct-custom' 'wps-office-ct-custom-fonts')
pkgver=11.8.2.12019.AK.preload.sw
pkgrel=1
pkgdesc="WPS Office Pro (Custom Enterprise Version) with multi-arch support and runtime fixes"
arch=('x86_64' 'aarch64')
url="https://www.wps.cn"
license=('custom')
options=('!strip' '!docs')

# 透過動態拼接 pkgver，實現自動從你的 GitHub Release 下載對應架構的原始包
source_x86_64=("https://github.com/Capricornus007/wps-office-ct-custom/releases/download/${pkgver}/UOS_amd64.deb")
source_aarch64=("https://github.com/Capricornus007/wps-office-ct-custom/releases/download/${pkgver}/UOS_arm64.deb")

# 由於是從遠端下載，強烈建議不要用 SKIP，直接填入你本地檔案的實際 SHA256 雜湊值（可用 sha256sum 檔案 獲取）
# 這樣 makepkg 下載完會自動校驗，防止下載損壞，也能確保安全性
sha256sums_x86_64=('33f135a831192589d1e554c04435572d31e1d5b39c9d3f4fcb8ff4d6c92bd2f1')
sha256sums_aarch64=('44120fe568c6802b530d5124437504680c8c97b42b6aef9bd2b357e3c81eab97')

prepare() {
  msg "正在根據架構 [${CARCH}] 解壓對應的 DEB 核心結構..."
  mkdir -p "${srcdir}/deb-extract"
  
  # 根據當前編譯架構，動態解壓對應的 data.tar.xz
  # $srcdir 會自動指向當前架構下載/放置的 data.tar.xz
  if [ -f "${srcdir}/data.tar.xz" ]; then
    tar -xf "${srcdir}/data.tar.xz" -C "${srcdir}/deb-extract"
  else
    error "找不到解壓所需的 data.tar.xz，請檢查本地 deb 是否完整。"
    return 1
  fi
}

# 1. 主程式包 (自動適應 x86_64 / aarch64)
package_wps-office-ct-custom() {
  conflicts=('wps-office' 'wps-office-365')
  provides=('wps-office')
  depends=('dbus' 'glu' 'gconf' 'libxss' 'gtk3' 'wps-office-ct-custom-fonts')

  cd "${srcdir}/deb-extract"

  # 建立基礎結構
  mkdir -p "${pkgdir}/opt/apps" \
           "${pkgdir}/usr/share/applications" \
           "${pkgdir}/usr/share/autostart" \
           "${pkgdir}/usr/share/icons" \
           "${pkgdir}/usr/bin" \
           "${pkgdir}/etc"

  msg "正在複製 WPS 主程式檔案（不含字型）..."
  cp -r opt/apps/cn.wps.wps-office-pro "${pkgdir}/opt/apps/"
  [ -d opt/kingsoft ] && cp -r opt/kingsoft "${pkgdir}/opt/"
  [ -d opt/.auth ] && cp -r opt/.auth "${pkgdir}/opt/"
  [ -d usr/share/mime ] && cp -r usr/share/mime "${pkgdir}/usr/share/"
  [ -d usr/lib ] && cp -r usr/lib "${pkgdir}/usr/"

  # 搬移及相容性處理 UOS 快捷方式與圖示
  local _entries="${pkgdir}/opt/apps/cn.wps.wps-office-pro/entries"
  [ -d "${_entries}/applications" ] && cp "${_entries}"/applications/*.desktop "${pkgdir}/usr/share/applications/"
  [ -d "${_entries}/autostart" ] && cp "${_entries}"/autostart/*.desktop "${pkgdir}/usr/share/autostart/"
  [ -d "${_entries}/icons/hicolor" ] && cp -r "${_entries}"/icons/hicolor/* "${pkgdir}/usr/share/icons/" 2>/dev/null || true

  # 修正功能表分類
  if compgen -G "${pkgdir}/usr/share/applications/*.desktop" > /dev/null; then
    sed -i 's|Categories=.*|&Office;|' "${pkgdir}/usr/share/applications"/*.desktop
  fi

  # 建立 /usr/bin 軟連結
  local _bin_path="/opt/apps/cn.wps.wps-office-pro/files/bin"
  ln -s "${_bin_path}/wps" "${pkgdir}/usr/bin/wps"
  ln -s "${_bin_path}/wpp" "${pkgdir}/usr/bin/wpp"
  ln -s "${_bin_path}/et" "${pkgdir}/usr/bin/et"
  ln -s "${_bin_path}/wpspdf" "${pkgdir}/usr/bin/wpspdf"

  # 核心相容性修正：注入環境變數與 libdbus 劫持
  for _cmd in wps wpp et wpspdf; do
    if [ -f "${pkgdir}${_bin_path}/${_cmd}" ]; then
      # 1. 解決 dbus 在 Arch 下裝瞎的問題
      sed -i '2i export LD_PRELOAD=/usr/lib/libdbus-1.so' "${pkgdir}${_bin_path}/${_cmd}"
      # 2. 注入 fcitx 輸入法環境變數
      sed -i '3i [[ "$XMODIFIERS" == "@im=fcitx" ]] && export QT_IM_MODULE=fcitx' "${pkgdir}${_bin_path}/${_cmd}"
    fi
  done

  # 清理老舊相容性崩潰在庫 (同時相容兩種架構下的 office6 位置)
  rm -f "${pkgdir}${_bin_path}/libcrypto.so"* 2>/dev/null || true
  rm -f "${pkgdir}/opt/apps/cn.wps.wps-office-pro/files/kingsoft/wps-office/office6/libstdc++.so.6"* 2>/dev/null || true
  # 預埋教育版完美授權配置，實現首次啟動即免啟用、免彈窗
  msg "正在預埋免啟用全域設定檔..."

  # 建立系統使用者預設配置目錄（Arch 標準路徑）
  mkdir -p "${pkgdir}/etc/skel/.config/Kingsoft"

  # 寫入 AuthInfo 完美過期時間
  cat << 'EOF' > "${pkgdir}/etc/skel/.config/Kingsoft/AuthInfo.conf"
[AuthInfo]
fld=0
ted=aa:8f:27:57
EOF

  # 寫入 WPSCloud 企業/授權強行宣告
  cat << 'EOF' > "${pkgdir}/etc/skel/.config/Kingsoft/WPSCloud.conf"
[General]
specific_companyintro=true
guestAccount=false

[Nse]
IsEnterprise=1

[license]
isLicensed=true
EOF

  msg "WPS 主程式包 [${CARCH}] 建置完成！"
}

# 2. 獨立字型包 (架構設為 any，因為字型檔案在各平台是通用的)
package_wps-office-ct-custom-fonts() {
  pkgdesc="Symbol and custom fonts bundled with WPS Office Pro"
  arch=('any')
  depends=('fontconfig')
  provides=('wps-office-fonts')
  conflicts=('wps-office-fonts')

  cd "${srcdir}/deb-extract"

  mkdir -p "${pkgdir}/usr/share/fonts"
  mkdir -p "${pkgdir}/etc/fonts"

  msg "正在將 WPS 內建字型抽離至獨立軟體包..."
  [ -d usr/share/fonts ] && cp -r usr/share/fonts/* "${pkgdir}/usr/share/fonts/"
  [ -d etc/fonts ] && cp -r etc/fonts/* "${pkgdir}/etc/fonts/"

  msg "WPS 獨立字型包建置完成！"
}
