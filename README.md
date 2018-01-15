## Day 28 - 邁向 DevOps 之路 (1)

### 本日共賞

* 系統架構
* 部署 Jenkins

### 希望你知道

* DevOps
* CI/CD
* [Day 27 - 漫步在雲端：在 GCP 中建立 k8s 叢集](https://ithelp.ithome.com.tw/articles/10193961)


既然這次是參加 DevOps 組別，勢必要與 DevOps 做個完美的結合。我們在過去的二十幾天內，一起探討了 k8s 的概念、各種不同的物件以及欣賞了各種不同的應用。最終，當然是希望將 k8s 套用到日常運作的系統內。[在 GCP 中建立 k8s 叢集](https://ithelp.ithome.com.tw/articles/10193961) 已經介紹過如何在 GCP 平台上建立 k8s 叢集，因此在這最後的時間，我們就以 GCP 當作例子示範來欣賞一下如何建立一條自動部署的 Pipeline。

#### 系統架構

讓我們先看一下整個系統的流程架構圖

![](https://ithelp.ithome.com.tw/upload/images/20180107/20107062tZLXYswsgi.png)

* `開發者`：就是你拉！你可以利用不同的程式語言，無論是 Java, Go, PHP, Node, Python, ... 等等，開發你的應用程式。
* `Code Repository`：當你在開發過程中，可以把開發紀錄存放在 Code Repository。無論是追縱、維護程式都有很多好處。常見的有 [github](https://github.com/), [bitbucket](https://bitbucket.org/), [Google Container Registry](https://cloud.google.com/container-registry/?hl=zh-tw),... 等等。

> 我們這次使用 Google Container Registry

* `CI/CD 工具`：當程式撰寫完成並上傳到 Code Repository 後，就輪到 CI/CD 工具上場了，透過適當的設定，這些工具會在 Repository 發生變化時，嘗試編譯、執行、測試或部署新版本的應用程式，常見的工具有 [Jenkins](https://jenkins-ci.org/), [Drone](https://github.com/drone/drone), [CircleCI](https://circleci.com/docs/1.0/continuous-deployment-with-google-container-engine/), ... 等等。

> 我們這次使用 Jenkins 

* `k8s 開發測試環境`：供內部測試使用，可與正式環境並存在 k8s 叢集中。

> 透過命名空間，我們可以在 k8s 叢集中切開正式環境與測試環境。好處是兩個環境不會互相干擾，但卻可以在相同叢集下運作。

* `k8s 正式環境`：提供對外的服務，可供一般使用者存取。

> 雖然上圖看起來正式與測試環境不同，但這裡提到的測試環境與正式環境是同時存在同一個 k8s 叢集中，當然如果需要，你想要分開兩個叢集也可以。


#### 部署 Jenkins

Google 有提供一個範例可供下載，首先打開 Cloud Shell 下載

> 忘記什麼是 Cloud Shell 了嗎？ 請參考 [Day 27 - 漫步在雲端：在 GCP 中建立 k8s 叢集](https://ithelp.ithome.com.tw/articles/10193961)

```bash
$ cd ~; git clone https://github.com/GoogleCloudPlatform/continuous-deployment-on-kubernetes.git
Cloning into 'continuous-deployment-on-kubernetes'...
remote: Counting objects: 690, done.
remote: Compressing objects: 100% (3/3), done.
remote: Total 690 (delta 2), reused 1 (delta 1), pack-reused 686
Receiving objects: 100% (690/690), 1.10 MiB | 338.00 KiB/s, done.
Resolving deltas: 100% (356/356), done.
```

你可以在 `continuous-deployment-on-kubernetes/jenkins/k8s/` 目錄下找到 Jenkins 相關的部署設定，包含

* `jenkins.yaml`：Deployment 相關設定
* `service_jenkins.yaml`：Service 相關設定
* `options`：供設定 Jenkins 預設密碼使用
* `lb/ingress.yaml`： Ingress 物件相關設定

> 我們就直接在 [漫步在雲端：在 GCP 中建立 k8s 叢集](https://ithelp.ithome.com.tw/articles/10193961) 建立的 `ithome` 叢集上部署，所以這邊就直接假設已經可以用 kubectl 操作 `ithome` 叢集。底下都是在 Cloud Shell 環境下操作

接著，下載已經設定好的 Jenkins 映像檔

```bash
$ gcloud compute images create jenkins-home-image --source-uri https://storage.googleapis.com/solutions-public-assets/jenkins-cd/jenkins-home-v3.tar.gz
Created [https://www.googleapis.com/compute/v1/projects/stately-magpie-188902/global/images/jenkins-home-image].
NAME                PROJECT                FAMILY  DEPRECATED  STATUS
jenkins-home-image  stately-magpie-188902                      READY
```

然後把 `jenkins-home-image` 轉成 Volume

```bash
$ gcloud compute disks create jenkins-home --image jenkins-home-image
Created [https://www.googleapis.com/compute/v1/projects/stately-magpie-188902/zones/asia-east1-b/disks/jenkins-home].
NAME          ZONE          SIZE_GB  TYPE         STATUS
jenkins-home  asia-east1-b  10       pd-standard  READY
```

> 這裡使用的名稱 `jenkins-home` 是直接參考 jenkins.yaml 內掛載 Volume 的名稱，另外我們直接使用範例而沒有對 Jenkins 的設定另外說明，如果想知道細節如何設定，可以參考 [為 k8s 配置 Jenkins](https://cloud.google.com/solutions/configuring-jenkins-kubernetes-engine)


接著，設定一下首次登入 Jenkins 需要用到的密碼，我們利用 `openssl` 建立一個隨機密碼，然後利用 `sed` 指令來替換 `options` 的內容

```bash
$ PASSWORD=`openssl rand -base64 8`; echo "Your password is $PASSWORD"; sed -i.bak s#CHANGE_ME#$PASSWORD# ~/continuous-deployment-on-kubernetes/jenkins/k8s/options
Your password is K2GUpvoJWDM=   <=== 這裡會印出登入 Jenkins 的密碼
```

接下來，建立一個 Jenkins 專屬的命名空間 `jenkins`，然後把剛剛產生的密碼以 Secret 的形式部署到從集中

```bash
$ kubectl create namespace jenkins   <=== 建立 jenkins 命名空間
namespace "jenkins" created

$ kubectl create secret generic jenkins --from-file=continuous-deployment-on-kubernetes/jenkins/k8s/options --namespace=jenkins
secret "jenkins" created
```

這樣部署前的準備算是完成了，接下來就可以直接部署 Jenkins

```bash
$ kubectl apply -f continuous-deployment-on-kubernetes/jenkins/k8s
deployment "jenkins" created
service "jenkins-ui" created
service "jenkins-discovery" created
```

> 這裡有沒有發現什麼不一樣？ 這次沒有一個一個部署，而是直接指定一個目錄。kubectl 會查看目錄下所有的 yaml 並一起部署到叢集中，是不是很方便！

接著，查看一下狀態

```bash
$ kubectl get pods --namespace=jenkins
NAME                      READY     STATUS    RESTARTS   AGE
jenkins-87c47bbb8-9h2fp   0/1       Pending   0          2m

$ kubectl get service --namespace=jenkins
NAME                TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
jenkins-discovery   ClusterIP   10.59.252.175   <none>        50000/TCP        2m
jenkins-ui          NodePort    10.59.250.182   <none>        8080:31358/TCP   2m
```

非常好！Jenkins 已經部署到 `ithome`！真的是這樣嗎？等了一下你就會發現 Pod 的狀態怎麼一直都停在 `Pending`，運氣真是太好了，馬上就發生問題！

根據 [k8s 不自賞 - 常見問題與建議](https://ithelp.ithome.com.tw/articles/10192193) 曾經提到的內容，讓我們查看一下發生什麼事情了！

```bash
$ kubectl describe pod jenkins-87c47bbb8-9h2fp --namespace jenkins
Events:
  Type     Reason            Age                From               Message
  ----     ------            ----               ----               -------
  Warning  FailedScheduling  31s (x26 over 6m)  default-scheduler  No nodes are available that match all of the predicates: Insufficient cpu (2).
```

> Pod 名稱要記得換掉喔！以你現在的程度應該不會犯這種錯了吧！

太神奇了，直接就告訴我們 `Insufficient cpu`，真是奇怪了，我們不是已經開了兩台主機嗎？沒關係，再查查

```bash
$ kubectl describe nodes
...
Allocated resources:
  (Total limits may be over 100 percent, i.e., overcommitted.)
  CPU Requests  CPU Limits  Memory Requests  Memory Limits
  ------------  ----------  ---------------  -------------
  708m (75%)    248m (26%)  741976Ki (27%)   1110616Ki (40%)
  
...
Allocated resources:
  (Total limits may be over 100 percent, i.e., overcommitted.)
  CPU Requests  CPU Limits  Memory Requests  Memory Limits
  ------------  ----------  ---------------  -------------
  480m (51%)    0 (0%)      320Mi (12%)      470Mi (17%)
...
```

這裡發現，這兩個 Node 已經分別被佔住 `75%` 與 `51%` 的 CPU 資源，那檢查一下 `~/continuous-deployment-on-kubernetes/jenkins/k8s/jenkins.yaml` 的內容

```
# jenkins.yaml

...
resources:
  limits:
    cpu: 500m
    memory: 1500Mi
  requests:
    cpu: 500m   <=== 看到這個，你應該知道發生什麼事了吧！
    memory: 1500Mi
...
```

根據 `jenkins.yaml` 的內容發現，原來 Jenkins 的部署要求 `500m` 的 CPU 資源。的確！以目前的叢集狀態，真的沒有一個 Node 有足夠的 CPU 資源能夠部署 Jenkins。因此，我們稍微修改一下 

```
# jenkins.yaml

...
resources:
  limits:
    cpu: 300m
    memory: 1200Mi
  requests:
    cpu: 300m   <=== 看到這個，你應該知道發生什麼事了吧！
    memory: 1200Mi
...
```

然後再部署一次，並查看狀態

```bash
$ kubectl apply -f continuous-deployment-on-kubernetes/jenkins/k8s
deployment "jenkins" configured
service "jenkins-ui" unchanged         <=== 沒有變更的不會影響
service "jenkins-discovery" unchanged  <=== 沒有變更的不會影響

$ kubectl get pods --namespace=jenkins
NAME                       READY     STATUS              RESTARTS   AGE
jenkins-576b94cf98-kj87l   0/1       ContainerCreating   0          36s
<=== 名稱已經變了

$ kubectl get pods --namespace=jenkins
NAME                       READY     STATUS    RESTARTS   AGE
jenkins-576b94cf98-kj87l   1/1       Running   0          1m
```

稍等一下，就會發現一切都正常運作了，接著照 [保護你的系統：套用 SSL](https://ithelp.ithome.com.tw/articles/10193959) 的方式替 Jenkins 加個 SSL 保護

```bash
$ openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout /tmp/tls.key -out /tmp/tls.crt -subj "/CN=jenkins/O=jenkins"
Generating a 2048 bit RSA private key
................................................................................+++
...........+++
writing new private key to '/tmp/tls.key'
-----

$ kubectl create secret generic tls --from-file=/tmp/tls.crt --from-file=/tmp/tls.key --namespace jenkins
secret "tls" created
```

最後一個步驟，部署 Ingress

```bash
$ kubectl apply -f continuous-deployment-on-kubernetes/jenkins/k8s/lb/ingress.yaml
ingress "jenkins" created
```

然後就等待 GKE 幫你處理後面的事情，最後 Ingress 會被分配一個 IP

```bash
$ kubectl get ingress --namespace jenkins
NAME      HOSTS     ADDRESS          PORTS     AGE
jenkins   *         35.227.250.195   80, 443   1m
```

接著你就可以打開瀏覽器輸入 `https://35.227.250.195` 登入 Jenkins，預設的帳號是 `jenkins` 而登入的密碼就是上面 `options` 的密碼。

![](https://ithelp.ithome.com.tw/upload/images/20180107/20107062Z2XTOEW2U2.png)

> 如果忘記密碼，可以再去 `continuous-deployment-on-kubernetes/jenkins/k8s/options` 查看


本文同步發表於 [https://jlptf.github.io/ironman2018-day28/](https://jlptf.github.io/ironman2018-day28/)

