# Airflow-learn
主要參照[一段Airflow與資料工程的故事：談如何用Python追漫畫連載](https://leemeng.tw/a-story-about-airflow-and-data-engineering-using-how-to-use-python-to-catch-up-with-latest-comics-as-an-example.html#app-v2)作為Airflow的入門，這篇文章已經講解得非常好，本文僅記錄實作過程中遇到的一些問題和解法。

## Windows下安裝Linux環境、Anaconda
到Microsoft Store當中搜尋Ubuntu 18.04 LTS版本，安裝後即可使用Linux terminal，開啟後就可以執行以下程式碼把整個專案複製下來：

```
git clone https://github.com/leemengtaiwan/airflow-tutorials.git
cd airflow-tutorials
```

由於Airflow是利用Python做操作，在這裡直接先透過以下指令下載Anaconda環境：

```
wget https://repo.anaconda.com/archive/Anaconda3-2020.07-Linux-x86_64.sh
```

接著再執行安裝：

```
bash Anaconda3-2020.07-Linux-x86_64.sh
```

安裝過程中需先閱讀完授權條款、更改安裝路徑、是否要執行conda init的設定等等，可以參考[Office指南](https://officeguide.cc/ubuntu-linux-install-anaconda-data-science-platform-tutorial)講解，然而未來在terminal中呼叫conda是否會直接載入base環境就單純依個人習慣而定。

在安裝完後須先執行以下指令，username則是填入安裝完Ubuntu時設定的使用者名稱：

```
export PATH=/home/<username>/anaconda3/bin:$PATH
source ~/.bashrc
```

基本上到這裡就可以呼叫conda檢查是否有正常執行，然而比較好的方式是寫進`~/.bashrc`裡面，這樣以後開啟terminal就會自動幫你做好source並且正常使用Anaconda，以上程式碼就可以改寫成「把`export PATH=/home/<username>/anaconda3/bin:$PATH`寫入`~/.bashrc`中」：

```
echo 'export PATH=/home/<username>/anaconda3/bin:$PATH' >> ~/.bashrc
```

當然我們也可以直接用`nano ~/.bashrc`開啟檔案後自行新增程式碼進去。

## 建立Airflow環境
依照教學中的指示執行以下程式碼，需要注意的一點是初始化Airflow Metadata DB的寫法目前版本有稍做更動：

```
conda create -n airflow-tutorials python=3.6 -y
source activate airflow-tutorials
pip install "apache-airflow[crypto, slack]"
export AIRFLOW_HOME="$(pwd)"
airflow db init
```

在上述程式碼中，有二點要特別注意：
  
* 如同上一小節所述，比較好的方式也是把`AIRFLOW_HOME`寫入`~/.bashrc`中，也就是`echo 'export AIRFLOW_HOME="$(pwd)"' >> ~/.bashrc`
  
* 若沒有設定`export AIRFLOW_HOME="$(pwd)"`就做`airflow db init`的話，那麼Airflow就會使用當初作者測試時使用的路徑，裡面會有很多的example可以供參考（也是很棒的教學參考），但就不會有你git clone下來的repo的路徑而發生找不到該文的教學檔案

此外，在Airflow環境第一次建立時會需要username或password等等的設定，執行以下程式碼並填入括弧內資訊即可，未來就會作為登入Airflow環境的帳號密碼：

```
airflow users create --role <role> --username <username> --email <email> --firstname <firstname> --lastname <lastname> --password <password>
```

都完成後就可以執行以下程式碼看到Airflow Web UI的介面啦！

```
airflow webserver -p 8080
```

## 建立Airflow工作
觀察該文中comic_app_v1和comic_app_v2的差異在哪裡，模組化是建立資料處理管線相當重要的概念。

建立Airflow tasks要將裡面寫的function獨立分開，如此一來當錯誤發生時就會止於特定function。假設你在跑一個已經建立好的Airflow資料管線時，某些function是很輕量的、而有些則是需要大計算量且須運行較長時間的，如果今天把所有資料處理邏輯都放在同一個function裡面，當其中執行任何一個輕量步驟失敗時，可能就會導致所有大計算量的步驟也得重新執行，此一來，重新再跑所耗費的成本就會很大。

## 測試Airflow工作
在我們放心讓Airflow幫我們排程comic_app_v2 DAG以前，必須養成好習慣先分別測試裡頭所有工作，確保它們的執行結果如我們預期：

```
airflow tasks test comic_app_v2 get_read_history 2018-01-01
取得使用者的閱讀紀錄

airflow tasks test comic_app_v2 check_comic_info 2018-01-01
跟紀錄比較，有沒有新連載？

airflow tasks test comic_app_v2 new_comic_available 2018-01-01
去漫畫網站看有沒有新的章節

...
```

假設做完所有工作的測試，想把comic_app_v2 DAG開始被Airflow排程，我們需要另外開啟Airflow排程器（Scheduler），再打開一個terminal並執行以下指令：

```
export AIRFLOW_HOME="$(pwd)"
source activate airflow-tutorials
airflow scheduler
```

在工作正式上線（Production）前，我們可以使用手動的方式去觸發DAG，你可以直接在Web UI上面直接把DAG給「On」起來，或在terminal中執行以下指令也可以達到相同效果：

```
airflow dags unpause comic_app_v2
airflow dags trigger_dag comic_app_v2
```

## 執行日期：排程最重要的概念
從Web UI可以看到剛剛手動觸發的comic_app_v2 DAG已經被Airflow排程器拿去產生一個新的DAG Run並成功執行。DAG跟DAG Run的差異在於：前者只是定義好的工作流程，後者則是該DAG在某個時間點實際被排程器拿去執行（Run）過後的結果，並會有一個執行日期（execute_date）。
    
此外，一個DAG Run中的執行日期，只等於它「負責」的日期，不等於它實際被Airflow排程器執行的日期，我們在以comic_app_v2為例：
    
```
default_args = {
    'owner': 'Meng Lee',
    'start_date': datetime(2100, 1, 1, 0, 0),
    'schedule_interval': '@daily',
    ...

with DAG('comic_app_v2', default_args=default_args) as dag:
    ...
```

這代表我們在1月1號23點59分結束以後，也就是1月2號0點的時候，將1月1號所有的使用者資料做彙總。一般而言，Airflow會`在start_date`加上一個`schedule_interval`之後開始第一次執行某個DAG，而該DAG Run的execute_date為`start_date`。

## 小結
Airflow擅長的是管理那些允許「事件發生時間」跟「實際數據處理時間」有落差的批次工作。因此Airflow都會在`start_date`加上`schedule_interval`長度的時間過完以後，才開始處理發生在`start_date`到`start_date+schedule_interval`之間的資料。

此外，在整個開發、建立、測試到上線Airflow工作前，必須養成以下好習慣：

* 在commic_app_v2.py中確認邏輯流程`>>`無誤（Upstream task & Downstream task）
* 在commic_app_v2.py中確認Airflow工作環境參數、訊息交換無誤（Xcom）
* 以python dags/commic_app_v2.py確保DAG定義無誤
* 利用Airflow tasks test指令分別測試每個Airflow工作執行如預期
* 使用Web UI點擊「Trigger Dag」按鈕或是透過airflow trigger來手動觸發DAG確認結果

## 注意事項
* 使用Airflow前須先`cd airflow-tutorials`設定路徑、`source ~/.bashrc`載入設定再`source activate airflow-tutorials`啟動Airflow環境。`source ~/.bashrc`中必須在airflow-tutorials的路徑下執行，這樣AIRFLOW_HOME的位置才會設定對，Airflow環境啟動時才會找的到Metadata DB裡去抓資料
* `sudo lsof -i tcp:8080`可以列出目前8080 port的使用狀態，`kill -9 <PID>`則可以刪除對應PID編號的程序

## Slack App設定
在這裡我們依照[一段Airflow與資料工程的故事：談如何用Python追漫畫連載](https://leemeng.tw/a-story-about-airflow-and-data-engineering-using-how-to-use-python-to-catch-up-with-latest-comics-as-an-example.html#app-v2)教學，同樣實作利用Slack做訊息更新的服務並以`comic_app_v3.py`作為架構去修改，改成一個追蹤Opensea感興趣NFT地板價（Floor price）的App。

首先，我們需要Selenium套件去做網頁訪問，因此先執行以下指令分別安裝Selenium套件：

```
pip install selenium
```

再下載Chrome driver並解壓縮：

```
sudo apt-get install unzip
wget -N http://chromedriver.storage.googleapis.com/100.0.4896.20/chromedriver_linux64.zip
unzip chromedriver_linux64.zip
```

我們把解壓縮後的Chrome driver移到`/usr/local/bin`並加入`~/.bashrc`中：

```
sudo mv -f ~/chromedriver /usr/local/bin
echp 'export PATH=$PATH:/usr/local/bin' >> ~/.bashrc
```

