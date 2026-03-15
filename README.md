beyler bu kod nasıl calışır anlatayım bu kod TERMUX, kali linux hepsinde çalışır tavsiyem kali linuxta kurun daha rahat kurulum aşağıda
# bypassbeast
WAF bypass payload generator for Kali Linux

## Kurulum

git clone https://github.com/Arda7362/bypassbeast.git
cd bypassbeast

python3 -m venv env
source env/bin/activate

pip install -r requirements.txt


## Kullanım

Temel kullanım:

python3 bypassbeast.py "<payload>"

Örnek:

python3 bypassbeast.py "<script>alert(1)</script>"


## WAF seçerek kullanım

python3 bypassbeast.py --waf cloudflare "<script>alert(1)</script>"

Desteklenen WAF türleri:

- cloudflare
- modsecurity
- imperva
- generic


## Agresif mod

python3 bypassbeast.py --aggressive "<script>alert(1)</script>"


