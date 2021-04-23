m3u8 是一種基於 HTTP Live Streaming 檔案影片格式，它主要是存放整個影片的基本資訊和分片(Segment)組成。

相信大家都看過m3u8格式檔案的內容，我們直來對比一下有什麼不同，然後教大家怎麼用python多程序實現下載並且合併。

# 一、兩者不同
- 非加密 的m3u8檔案
![](https://p1-tt-ipv6.byteimg.com/large/pgc-image/45d6271f2d524cbc8aa278da85562bab)
- 加密 的m3u8檔案
![](https://p1-tt-ipv6.byteimg.com/large/pgc-image/b0b2cb0e595c4fd5acb0163820307f86)

相信眼尖的小夥伴已經看出了2個內容的不同之處，對的，其實區別就在加密檔案的第 5 行的 #EXT-X-KEY 的資訊
這個資訊就是用來影片內容解密的，其實裡面的內容大多是一段字串，其實也就是解密時候的KEY值
那麼這個怎麼去解密呢，我們暫時不管，我們先來解釋一下每行的意思

第一行: #EXTM3U 宣告這是一個m3u8的檔案

第二行: #EXT-X-VERSION 協議的版本號

第三行: #EXT-X-MEDIA-SEQUENCE 每一個media URI 在 PlayList中只有唯一的序號，相鄰之間序號+1

第四行: #EXT-X-KEY 記錄了加密的方式，一般是AES-128以及加密的KEY資訊

第五行: #EXTINF 表示這段影片碎片的持續時間有多久

第六行: sA3LRa6g.ts 影片片段的名稱，獲取的時候需要拼接上域名，找到檔案的正確的路徑

# 二、爬蟲內容詳解
初始化m3u8下載類

`if name == "main": M3u8().hello().run()`

hello方法

```python
def hello(self): 
''' 
This is a welcome speech 
:return: self 
''' 
print("" 50) 
print(' ' 15 + 'm3u8連結下載小助手') 
print(' ' 5 + '作者: Felix Date: 2020-05-20 13:14') 
print(' ' 10 + '適用於非加密 | 加密連結') 
print("" * 50) 
return self
```

hello方法其實就是歡迎語，介紹了一些基本資訊

如果鏈式呼叫的話，必須返回 self，初學者需要注意

run方法

```python
def run(self): 
''' 
program entry, Input basic information 
''' 
downPath = str(input("碎片的儲存路徑, 預設./Download：")) or "./Download" 
savePath = str(input("影片的儲存路徑, 預設./Complete：")) or "./Complete" 
clearDebris = bool(input("是否清除碎片, 預設True：")) or True 
saveSuffix = str(input("影片格式, 預設ts：")) or "ts"

def parse(self, url): 
''' 
Analyze a link of m3u8 
:param url: string, the link need to analyze 
:return: list 
''' 
container = list() 
response = self.request(url).text.split('\n') 
for ts in response: 
    if '.ts' in ts: 
        container.append(ts) 
    if '#EXT-X-KEY:' in ts: 
        self.encrypt = True 
    return container
```
請求連結，判斷是否是加密m3u8還是非加密

將所有碎片檔案進行返回

開啟多程序，開啟程序池，加速下載速度

run processing to do something

```python
print('---程序開始執行...') 
po = multiprocessing.Pool(30) 
queue = multiprocessing.Manager().Queue()
size = 0 
for file in container: 
    sort = str(size).zfill(5) 
    po.apply_async(self.download, args=(queue, sort, file, downPath, url,)) 
    size += 1

po.close()
```

`zfill`方法，其實就是在數字前填充0，因為我希望下載的檔案是00001.ts，00002.ts這樣有序的，最後合併的時候才不會混亂

`queue` 是多程序共享變數的一種方式，用來顯示下載的進度條

download方法

```python
def download(self, queue, sort, file, downPath, url): 
''' 
Download the debris of video 
:param queue: the queue 
:param sort: which number debris 
:param file: the link of debris 
:param downPath: the path to save debris 
:param url: the link of m3u8 :return: None 
''' 

queue.put(file)

baseUrl = '/'.join(url.split("/")[:-1])

if self.encrypt:
    self.encryptKey = self.getEncryptKey(baseUrl)

if not file.startswith("http"):
file = baseUrl + '/' +file

debrisName = "{}/{}.ts".format(downPath, sort)

if not os.path.exists(debrisName):
    response = self.request(file)
    with open(debrisName, "wb") as f:
        if self.encrypt:
            data = self.aesDecode(response.content, self.encryptKey)
            f.write(data)
            f.flush()
        else:
            f.write(response.content)
            f.flush()
```
一開始就加入佇列，是為了防止檔案之前已經存在的情況下，導致長度不對

如果是加密m3u8就通過 getEncryptKey 去獲取KEY值

寫入檔案的時候如果是加密的，就將檔案進行 aesDecode 方法解密，具體請看原始碼

進度條顯示

```python
def progressBar(self, queue, count): 
''' 
Show progress bar 
:param queue: the queue 
:param count: the number count of debris 
:return: None 
''' 
print('---一共{}個碎片...'.format(count)) 
offset = 0 
while True: 
	offset += 1 
	file = queue.get() 
	rate = offset * 100 / count 
	print("\r%s下載成功，當前進度%0.2f%%, 第%s/%s個" % (file, rate, offset, count)) 
	if offset >= count: break
```

其實就是通過當前的下載到第幾個碎片，和所有碎片的數量進行比較

一旦大於等於總數的時候，就退出迴圈

