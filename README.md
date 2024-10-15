# ubuntu24.04でautofs
/mnt/hddにハードディスクのドライブ(ntfs)をマウントするケース。

## インストール
  ```
  sudo apt install autofs
  ```
## 設定
### /etc/auto.master
以下を末尾に追加。改行も追加したほうがいい。
  ```
  /mnt /etc/auto.hdd

  ```

### /etc/auto.hdd
以下の内容で作成。sda1のところは再起動で変わるかもしれないため、本来はUUIDを使うなどしたほうが良い。
   ```
   hdd    -fstype=ntfs   :/dev/sda1
   ```

## autofsを再起動 & アクセス確認
  ```
  sudo systemctl restart autofs
  ls /mnt/hdd
  ```
## 参考
https://wiki.archlinux.jp/index.php/Autofs
