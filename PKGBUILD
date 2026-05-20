# Maintainer: Sī Hàogāng <your_email@example.com>
# Description: WPS Office 2023 China Telecom Custom Pro Version (UOS Edition)

pkgname=wps-office-ct-custom
pkgver=11.8.2.12019.AK.preload.sw
pkgrel=1
pkgdesc="WPS Office 2023 (China Telecom Enterprise Custom Edition for UOS)"
arch=('x86_64' 'aarch64')
url="https://www.wps.cn"
license=('custom:Kingsoft')
# 根據 control 檔案修正後的精確依賴列表
depends=('fontconfig' 'libxrender' 'xdg-utils' 'glu' 'libpulse' 'libxss' 'sqlite' 'libtool' 'libtiff' 'libxslt' 'libjpeg-turbo' 'libpng' 'freetype2' 'bzip2')
options=(!strip !zipman !debug)

source_x86_64=("https://dl-r2.nxtrace.org/wps_dist/UOS_amd64.deb")
source_aarch64=("https://dl-r2.nxtrace.org/wps_dist/UOS_arm64.deb")
sha256sums_x86_64=('SKIP')
sha256sums_aarch64=('SKIP')

pkgver() {
  cd "${srcdir}"
  # 由於 makepkg 已經用 bsdtar 把 control.tar.xz 解壓到 srcdir 了，我們直接原地解壓它
  mkdir -p control-extract
  tar -xf control.tar.xz -C control-extract ./control 2>/dev/null || tar -xf control.tar -C control-extract ./control 2>/dev/null
  sed -n 's/^Version: //p' control-extract/control | tr '-' '.'
}

prepare() {
  cd "${srcdir}"
  
  msg "偵測到 makepkg 已自動拆解 deb。正在將 data 核心資料解壓到指定目錄..."
  mkdir -p "${srcdir}/deb-extract"
  
  # 判斷 data 壓縮包的實際格式並將內容「精確解壓」到 deb-extract 內
  if [ -f data.tar.xz ]; then
    tar -xf data.tar.xz -C "${srcdir}/deb-extract"
  elif [ -f data.tar.zst ]; then
    tar -xf data.tar.zst -C "${srcdir}/deb-extract"
  elif [ -f data.tar ]; then
    tar -xf data.tar -C "${srcdir}/deb-extract"
  else
    error "未找到 data.tar.* 資料包，請檢查 deb 是否完整！"
    return 1
  fi
}

package() {
  conflicts=('wps-office' 'wps-office-365')
  provides=('wps-office')

  # 切換到已經完全鋪平、露出 opt/ 和 usr/ 的精確目錄
  cd "${srcdir}/deb-extract"

  # 強制建立 Arch 系統目錄
  mkdir -p "${pkgdir}/opt/apps"
  mkdir -p "${pkgdir}/usr/share/applications"
  mkdir -p "${pkgdir}/usr/share/autostart"
  mkdir -p "${pkgdir}/usr/share/icons"
  mkdir -p "${pkgdir}/usr/bin"
  mkdir -p "${pkgdir}/etc"

  msg "開始複製 WPS 主程式檔案到軟體包..."
  
  # 1. 複製主程式與配置目錄
  if [ -d opt/apps/cn.wps.wps-office-pro ]; then
    cp -r opt/apps/cn.wps.wps-office-pro "${pkgdir}/opt/apps/"
  else
    error "錯誤：在解壓目錄中仍未找到 opt/apps/cn.wps.wps-office-pro！"
    return 1
  fi
  
  [ -d opt/kingsoft ] && cp -r opt/kingsoft "${pkgdir}/opt/"
  [ -d opt/.auth ] && cp -r opt/.auth "${pkgdir}/opt/"

  # 2. 複製系統配置、字型與庫
  [ -d usr/share/mime ] && cp -r usr/share/mime "${pkgdir}/usr/share/"
  [ -d usr/share/fonts ] && cp -r usr/share/fonts "${pkgdir}/usr/share/"
  [ -d usr/lib ] && cp -r usr/lib "${pkgdir}/usr/"
  [ -d etc/fonts ] && cp -r etc/fonts "${pkgdir}/etc/"

  msg "正在處理 UOS 桌面快捷方式與圖示..."
  # 3. 從已複製的目標中挪移桌面啟動器
  local _entries="${pkgdir}/opt/apps/cn.wps.wps-office-pro/entries"
  
  if [ -d "${_entries}/applications" ]; then
    cp "${_entries}"/applications/*.desktop "${pkgdir}/usr/share/applications/"
  fi
  if [ -d "${_entries}/autostart" ]; then
    cp "${_entries}"/autostart/*.desktop "${pkgdir}/usr/share/autostart/"
  fi
  if [ -d "${_entries}/icons/hicolor" ]; then
    cp -r "${_entries}"/icons/hicolor/* "${pkgdir}/usr/share/icons/" 2>/dev/null || true
  fi

  # 4. 修正功能表分類
  if compgen -G "${pkgdir}/usr/share/applications/*.desktop" > /dev/null; then
    sed -i 's|Categories=.*|&Office;|' "${pkgdir}/usr/share/applications"/*.desktop
  fi

  msg "正在建立 /usr/bin 軟連結與相容性修正..."
  # 5. 建立 /usr/bin 軟連結
  local _bin_path="/opt/apps/cn.wps.wps-office-pro/files/bin"
  ln -s "${_bin_path}/wps" "${pkgdir}/usr/bin/wps"
  ln -s "${_bin_path}/wpp" "${pkgdir}/usr/bin/wpp"
  ln -s "${_bin_path}/et" "${pkgdir}/usr/bin/et"
  ln -s "${_bin_path}/wpspdf" "${pkgdir}/usr/bin/wpspdf"

  # 6. 核心相容性修正：注入環境變數與 libdbus 劫持
  for _cmd in wps wpp et wpspdf; do
    if [ -f "${pkgdir}${_bin_path}/${_cmd}" ]; then
      # 優先注入 libdbus 修正，解決路徑相容性問題
      sed -i '2i export LD_PRELOAD=/usr/lib/libdbus-1.so' "${pkgdir}${_bin_path}/${_cmd}"
      # 注入 fcitx 輸入法環境變數
      sed -i '3i [[ "$XMODIFIERS" == "@im=fcitx" ]] && export QT_IM_MODULE=fcitx' "${pkgdir}${_bin_path}/${_cmd}"
    fi
  done

  # 7. 安全清理：刨除所有會引發 Arch 系統相容性崩潰的老舊自帶動態庫
  rm -f "${pkgdir}${_bin_path}/libcrypto.so"* 2>/dev/null || true
  
  # 👈 核心修正：徹底刪除自帶的老舊 C++ 標準庫，強迫 wps 使用 Arch 系統的 libstdc++.so.6
  rm -f "${pkgdir}/opt/apps/cn.wps.wps-office-pro/files/kingsoft/wps-office/office6/libstdc++.so.6"* 2>/dev/null || true
  
  msg "WPS 政企版本地原生包打包完成！"
}
