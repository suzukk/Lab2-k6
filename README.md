# B232270011 С.Тэргэл Лаборатор 2

# Гүйцэтгэлийн хэмжүүрийг Grafana k6-аар хэмжих

Энэхүү лабораторийн ажлаар Grafana k6 ашиглан веб системийн гүйцэтгэлийг хэмжиж үзсэн. Туршилтын явцад latency буюу хүсэлтийн хариу өгөх хугацаа, throughput буюу секундэд боловсруулж байгаа хүсэлтийн тоо, error rate буюу алдааны хувь зэргийг хэмжсэн. Мөн хэрэглэгчийн ачааллыг 5, 30, 100 VU болгон нэмэгдүүлж системийн гүйцэтгэл хэрхэн өөрчлөгдөж байгааг харьцуулсан. Туршилтад лабораторийн зааварт дурдсан `https://test.k6.io` дадлагын сайтыг ашигласан.

================ Алхам 1 (k6 орчин бэлдэх) ==================

Эхлээд ашиглаж байгаа Arch Linux үйлдлийн систем дээр Grafana k6 суулгах шаардлагатай болсон.

Анх Arch Linux-ийн package manager болох pacman болон AUR-аас k6 хайж үзсэн боловч шууд олдсонгүй.

```bash
pacman -Ss '^k6$'
yay -Ss '^k6$'
```

Иймээс Grafana k6-ийн албан ёсны release хувилбарыг ашиглан суулгасан. Миний компьютер `x86_64` архитектуртай байсан тул Linux amd64 хувилбарыг татаж авсан.

```bash
uname -m
```

Үр дүн:

```text
x86_64
```

k6-ийн binary-г татаж аваад хэрэглэгчийн local bin folder-т байрлуулсан.

```bash
mkdir -p ~/.local/bin

curl -L https://github.com/grafana/k6/releases/download/v2.2.0/k6-v2.2.0-linux-amd64.tar.gz -o /tmp/k6.tar.gz

tar -xzf /tmp/k6.tar.gz -C /tmp

cp /tmp/k6-v2.2.0-linux-amd64/k6 ~/.local/bin/k6

chmod +x ~/.local/bin/k6
```

Дараа нь `~/.local/bin`-ийг PATH-д нэмсэн.

```bash
echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc
```

k6 суусан эсэхийг шалгасан.

```bash
k6 version
```

Үр дүн:

```text
k6 v2.2.0 (commit/00a9a1b7f5, go1.26.5, linux/amd64)
```

Ингэснээр Arch Linux орчинд k6-ийг амжилттай тохируулсан.

================ Алхам 2 (Анхны k6 test script) ==================

Дараа нь лабораторийн тестийг хийхийн тулд `k6-lab2` folder үүсгэсэн.

```bash
mkdir -p ~/k6-lab2
cd ~/k6-lab2
```

`script.js` файл үүсгээд 5 VU, 30 секундын baseline тест бичсэн.

```javascript
import http from 'k6/http';
import { sleep, check } from 'k6';

export const options = {
  vus: 5,
  duration: '30s',
};

export default function () {
  const res = http.get('https://test.k6.io');

  check(res, {
    'status 200 байна': (r) => r.status === 200,
  });

  sleep(1);
}
```


Тестийг:

```bash
k6 run script.js
```

командаар ажиллуулсан.


================ Алхам 3 (5 VU baseline хэмжилт) ==================

Лабораторийн дараагийн ачааллын тестүүдийг харьцуулахын тулд эхлээд 5 VU baseline тестийн үр дүнг хадгалсан.

```bash
mkdir -p results
k6 run script.js | tee results/run-05vu.txt
```

Baseline тестийн үр дүн:

```text
http_req_duration:
avg=174.63ms
p(90)=277.69ms
p(95)=318.4ms

http_req_failed:
0.00%

http_reqs:
214
6.896345/s
```

Ингэснээр baseline-ийн p95 latency нь **318.40 ms**, throughput нь **6.896 req/s**, error rate нь **0.00%** болсон.

Энэ baseline-ийн p95 утгыг дараагийн SLO threshold тодорхойлоход ашигласан.

================ Алхам 4 (5, 30, 100 VU ачааллын тест) ==================

Ачаалал нэмэгдэхэд системийн гүйцэтгэл хэрхэн өөрчлөгдөж байгааг харахын тулд 5, 30, 100 VU дээр тус тус 1 минутын тест ажиллуулсан.

### 5 VU

```bash
k6 run --vus 5 --duration 1m script.js | tee results/run-05vu.txt
```

### 30 VU

```bash
k6 run --vus 30 --duration 1m script.js | tee results/run-30vu.txt
```

30 VU тестийн үр дүн:

```text
http_req_duration:
p(90)=242.69ms
p(95)=273.1ms

http_req_failed:
0.00%

http_reqs:
2726
44.592604/s
```

### 100 VU

```bash
k6 run --vus 100 --duration 1m script.js | tee results/run-100vu.txt
```

100 VU тестийн үр дүн:

```text
http_req_duration:
p(90)=225.29ms
p(95)=226.52ms

http_req_failed:
0.00%

http_reqs:
9214
150.386495/s
```

Ачааллын тестүүдийн харьцуулалт:

| VUs |       p90 |       p95 |    Throughput | Error rate |
| --: | --------: | --------: | ------------: | ---------: |
|   5 | 277.69 ms | 318.40 ms |   6.896 req/s |      0.00% |
|  30 | 242.69 ms | 273.10 ms |  44.593 req/s |      0.00% |
| 100 | 225.29 ms | 226.52 ms | 150.386 req/s |      0.00% |

Энд VU-ийн тоо нэмэгдэхэд throughput маш их нэмэгдсэн. 5 VU үед 6.896 req/s байсан бол 100 VU үед 150.386 req/s болсон. Харин latency нь VU нэмэгдэх бүрд өсөөгүй, харин энэ туршилтаар p95 буурсан үр дүн гарсан. Үүнийг систем заавал ачаалал нэмэгдэхэд хурдсана гэж дүгнэж болохгүй `test.k6.io` нь алсын сервер учраас сүлжээ болон серверийн тухайн үеийн үр дүнд хамаарж болно.

Бүх гурван тестийн error rate **0.00%** байсан бөгөөд бүх check 100% амжилттай болсон.

================ Алхам 5 (Stages ашиглан ачаалал өөрчлөх) ==================

Дараа нь хэрэглэгчийн ачааллыг нэг дор засах биш үе шаттайгаар нэмэгдүүлж, бууруулахын тулд `stages.js` файл үүсгэсэн.

```javascript
import http from 'k6/http';
import { sleep, check } from 'k6';

export const options = {
  stages: [
    { duration: '30s', target: 5 },
    { duration: '1m', target: 30 },
    { duration: '30s', target: 100 },
    { duration: '30s', target: 0 },
  ],
};

export default function () {
  const res = http.get('https://test.k6.io');

  check(res, {
    'status 200 байна': (r) => r.status === 200,
  });

  sleep(1);
}
```

Энэ тестээр эхлээд 5 VU хүртэл ихэсгэж, дараа нь 30 VU, дараа нь 100 VU хүртэл ихэсгээд эцэст нь 0 VU болгож бууруулсан.

Тестийг:

```bash
k6 run stages.js | tee results/stages.txt
```

командаар ажиллуулсан.

Нийт тестийн хугацаа 2 минут 30 секунд байсан.

Үр дүн:

```text
checks_total: 3558
checks_succeeded: 100.00%

p(90)=224.86ms
p(95)=226.53ms

http_req_failed:
0.00%

http_reqs:
7116
47.149269/s

vus_max:
100
```

Stages тестээр нийтдээ 100 VU хүртэл ачааллыг нэмэгдүүлж үзсэн боловч ямар ч HTTP error гараагүй.

================ Алхам 6 (SLO threshold тодорхойлох) ==================

Лабораторийн шаардлагын дагуу threshold-ийг шууд дурын тоогоор сонгох биш baseline тестийн үр дүнд үндэслэн тодорхойлсон.

Baseline-ийн p95:

```text
318.40 ms
```

Үүнийг 1.5-аар үржүүлсэн:

```text
318.40 × 1.5 = 477.60 ms
```

Иймээс performance SLO-г:

```text
p(95) < 477.6 ms
```

гэж тодорхойлсон.

error rate:

```text
rate < 0.01
```

буюу 1%-иас бага байх шаардлага тавьсан.

`threshold-pass.js` файлд:

```javascript
thresholds: {
  http_req_duration: ['p(95)<477.6'],
  http_req_failed: ['rate<0.01'],
},
```

гэж тохируулсан.

PASS тестийн үр дүн:

```text
http_req_duration
✓ 'p(95)<477.6' p(95)=275.08ms

http_req_failed
✓ 'rate<0.01' rate=0.00%
```

275.08 ms нь 477.6 ms-ээс бага байсан тул performance SLO амжилттай биелсэн.

```text
275.08 ms < 477.6 ms
```

PASS тестийн бүрэн output:

```text
results/threshold-pass.txt
```

================ Алхам 7 (Threshold FAIL тест) ==================

Threshold ажиллаж байгаа эсэхийг шалгахын тулд зориудаар маш хатуу threshold тавьж FAIL тест хийсэн.

Энэ тестэд:

```javascript
thresholds: {
  http_req_duration: ['p(95)<50'],
  http_req_failed: ['rate<0.01'],
},
```

гэж тохируулсан.

Тестийн бодит p95:

```text
279.94 ms
```

Харин шаардлага нь:

```text
p(95) < 50 ms
```

байсан.

Тиймээс:

```text
279.94 ms > 50 ms
```

учраас threshold FAIL болсон.

Үр дүн:

```text
http_req_duration
✗ 'p(95)<50' p(95)=279.94ms

http_req_failed
✓ 'rate<0.01' rate=0.00%
```

Энэ нь threshold-ийн шаардлага хангагдаагүй үед k6 тухайн threshold-ийг FAIL гэж тэмдэглэж байгааг харуулсан.

FAIL тестийн бүрэн output:

```text
results/threshold-fail.txt
```

================ Алхам 8 (Git болон GitHub) ==================

Лабораторийн ажлын файлуудаа Git ашиглан version control хийж, GitHub repository-д байршуулсан.

GitHub repository:

```text
https://github.com/suzukk/Lab2-k6
```

Ажлын явцад 5 commit хийсэн.

```text
cd882da SLO threshold pass and fail tests
6867cae Staged test
00b7249 100 VU test results
33a82f5 30 VU test results
1e95226 Add initial k6 baseline test
```

GitHub руу SSH authentication ашиглан push хийсэн.

================ Алхам 9 (Үр дүнгийн файлууд) ==================

Туршилтын бүрэн output-уудыг `results` folder дотор хадгалсан.

```text
results/
├── run-05vu.txt
├── run-30vu.txt
├── run-100vu.txt
├── stages.txt
├── threshold-pass.txt
└── threshold-fail.txt
```

Эдгээр файлд k6-ийн бүтэн summary болон туршилтын үр дүнг хадгалсан тул README дээрх тоонуудыг эдгээр output файлуудаас шалгах боломжтой.

================ Дүгнэлт ==================

Энэхүү лабораторийн ажлаар Grafana k6 ашиглан веб системийн гүйцэтгэлийг хэмжиж сурсан. Эхлээд Arch Linux орчинд k6 v2.2.0-ийг суулгаж, `test.k6.io` сайт дээр анхны baseline тест хийсэн. 5 VU baseline тестээр p95 latency 318.40 ms, throughput 6.896 req/s, error rate 0.00% гарсан. Дараа нь 30 болон 100 VU ачаалал өгч үзэхэд throughput 44.593 req/s болон 150.386 req/s хүртэл нэмэгдсэн. Энэ туршилтаар ачаалал нэмэгдэхэд latency заавал ихэсэхгүй ба алсын сервер болон сүлжээний байдлаас хамаарж болохыг ойлгосон. Stages ашигласнаар ачааллыг 5 VU-ээс 30 VU, дараа нь 100 VU хүртэл нэмэгдүүлж, буцаан 0 VU болгож системийн гүйцэтгэлийг үе шаттай ажигласан. Baseline p95 утга дээр үндэслэн SLO threshold-ийг 477.60 ms гэж тодорхойлсон. PASS threshold тестээр p95 275.08 ms гарсан тул SLO шаардлагыг хангасан. Харин зориудаар 50 ms гэсэн хатуу threshold тавихад p95 279.94 ms болж FAIL гарсан нь threshold механизм зөв ажиллаж байгааг харуулсан. Ингэснээр k6 ашиглан latency, throughput, error rate, stages болон SLO threshold-ийг бодит туршилтаар хэмжиж, үр дүнг харьцуулж сурсан.
