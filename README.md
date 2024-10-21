# ubuntu24.04でautofs
/mnt/hddにハードディスクのドライブ(ntfs)をマウントするケース。

### インストール
  ```
  sudo apt install autofs
  ```
### 設定
#### /etc/auto.master
以下を末尾に追加。改行も追加したほうがいい。
  ```
  /mnt /etc/auto.map

  ```

#### /etc/auto.map
以下の内容で作成。sda1のところは再起動で変わるかもしれないため、本来はUUIDを使うなどしたほうが良い。
   ```
   hdd    -fstype=ntfs   :/dev/sda1
   ```

#### /etc/nsswitch.conf
autofsを名前解決をfilesの指定に変更する。これは必要か未確認。
  ```
  (省略)
  #automount:  sss
  automount:      files
  ```

### autofsを再起動 & アクセス確認
  ```
  sudo systemctl restart autofs
  ls /mnt/hdd
  ```

### その他
家のbuffaloのNASをマウントするときは/etc/auto.mapに以下の指定をした。  
この設定で/mnt/nasにアクセスすると自動マウントされる。
  ```
  nas  -fstype=cifs,username=<ユーザ名>,password=<パスワード>,vers=1.0 ://<IPアドレス>/<共有パス>
  ```
### 参考
https://wiki.archlinux.jp/index.php/Autofs

  
  
## WindowsでCPU、メモリ消費量モニター(1秒おき)
以下で表示できるが、CPU時間なのでクロックで割り算要。

```
$processId = <監視プロセスのプロセスID>

while ($true) {
     # 現在時刻を取得し、指定したフォーマットで表示
     $timestamp = Get-Date -Format "yyyy/MM/dd HH:mm:ss"

     # プロセスの情報を取得
     $process = Get-Process -Id $processId

     # 結果を出力
     Write-Host "$timestamp : CPU: $($process.CPU), MEM: $($process.WorkingSet) bytes"

     # 1秒間待機
     Start-Sleep -Seconds 1
}
```
