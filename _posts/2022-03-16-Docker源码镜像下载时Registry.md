---
layout: post
categories: [Docker]
description: none
keywords: Docker
---
# Docker源码


在api/server/router/local/local.go中有请求命令和调用的相应handler的对应关系，pull命令对应的语句是
```
NewPostRoute("/images/create", r.postImagesCreate)
```
调用的postImagesCreate函数的具体实现位于api/server/router/local/image.go中，该函数首先解析出要下载的image和tag、authConfig，之后建立imagePullConfig，该结构体的实现如下：
```
// ImagePullConfig stores pull configuration.
type ImagePullConfig struct {
    // MetaHeaders stores HTTP headers with metadata about the image
    // (DockerHeaders with prefix X-Meta- in the request).
    MetaHeaders map[string][]string
    // AuthConfig holds authentication credentials for authenticating with
    // the registry.
    AuthConfig *cliconfig.AuthConfig
    // OutStream is the output writer for showing the status of the pull
    // operation.
    OutStream io.Writer
}
```
可见，其中包含了与registry交互的参数，然后调用如下函数：
```
err = s.daemon.PullImage(image, tag, imagePullConfig)
```
PullImage函数的实现位于docker/daemon/daemon.go中，函数只有一条语句，调用了docker/graph/pull.go中的Pull函数，该函数中，从上到下比较重要的语句有：
```
repoInfo, err := s.registryService.ResolveRepository(image)
```
该函数将image名转化成*RepositoryInfo结构变量，该结构完成描述了一个repository
```
endpoints, err := s.registryService.LookupPullEndpoints(repoInfo.CanonicalName)
```
该函数返回APIEndpoint类型的数组，APIEndpoint结构如下：
```
type APIEndpoint struct {
    Mirror        bool
    URL           string
    Version       APIVersion
    Official      bool
    TrimHostname  bool
    TLSConfig     *tls.Config
    VersionHeader string
    Versions      []auth.APIVersion
}
```
可见，该结构可以表示一个与registry端连接的配置。接下来是一个循环，执行具体的下载镜像的操作，循环地从所有的注册的registry的endpoint中下载需要的镜像，其实现在1.6版本以上用到的仅仅是v2，代码如下：
```
for _, endpoint := range endpoints {
        logrus.Debugf("Trying to pull %s from %s %s", repoInfo.LocalName, endpoint.URL, endpoint.Version)

        if !endpoint.Mirror && (endpoint.Official || endpoint.Version == registry.APIVersion2) {
            if repoInfo.Official {
                s.trustService.UpdateBase()
            }
        }

        puller, err := NewPuller(s, endpoint, repoInfo, imagePullConfig, sf)
        if err != nil {
            lastErr = err
            continue
        }
        if fallback, err := puller.Pull(tag); err != nil {
            if fallback {
                if _, ok := err.(registry.ErrNoSupport); !ok {
                    discardNoSupportErrors = true
                    // save the current error
                    lastErr = err
                } else if !discardNoSupportErrors {
                    lastErr = err
                }
                continue
            }
            logrus.Debugf("Not continuing with error: %v", err)
            return err
}
```
该循环遍历所有的registry下载指定的镜像，NewPuller函数根据endpoint的Version类型决定返回的是1或者2版本的Puller结构，当registry为version：2版本是返回的结构如下：
```
type v2Puller struct {
    *TagStore
    endpoint  registry.APIEndpoint
    config    *ImagePullConfig
    sf        *streamformatter.StreamFormatter
    repoInfo  *registry.RepositoryInfo
    repo      distribution.Repository
    sessionID string
}
```
接下来调用v2Puller对应的Pull函数，该函数的实现位于docker/graph/pull_v2.go中，首先调用了NewV2Repository函数，建立了一个提供身份验证的http传输通道，并返回一个repository，该函数实现的功能大致上相当于registry：v1版本的NewSession函数，执行该函数后就能在建立的连接的基础上进行具体景象以及元数据的传输了。在NewV2Repository函数中，调用NewTransport函数，返回一个Transportde的结构，用于持续连接；之后调用http.NewRequest函数，建立一个跟对于特定url的请求；之后调用
http.Client.Do函数，处理自定义的request操作并返回相应的response；之后会对连接参数做设定，最后返回一个包含连接所有参数的distribution.Repository结构。
再回到Pull函数，如下：
```
func (p *v2Puller) Pull(tag string) (fallback bool, err error) {
    // TODO(tiborvass): was ReceiveTimeout
    p.repo, err = NewV2Repository(p.repoInfo, p.endpoint, p.config.MetaHeaders, p.config.AuthConfig, "pull")
    if err != nil {
        logrus.Debugf("Error getting v2 registry: %v", err)
        return true, err
    }

    p.sessionID = stringid.GenerateRandomID()

    if err := p.pullV2Repository(tag); err != nil {
        if registry.ContinueOnError(err) {
            logrus.Debugf("Error trying v2 registry: %v", err)
            return true, err
        }
        return false, err
    }
    return false, nil
}
```
在NewV2Repository建立的连接并返回的distribution.Repository结构p.repo基础上，调用了pullV2Repository函数，在该函数中，
```
manSvc, err := p.repo.Manifests(context.Background())
        if err != nil {
            return err
        }

        tags, err = manSvc.Tags()
        if err != nil {
            return err
        }
```
返回一个针对于特定repository的一个manifest的服务，Manifests函数是distribution中Repository结构中的一个函数，Repository定义在distribution/registry.go中，如下所示：
```
// Repository is a named collection of manifests and layers.
type Repository interface {
    // Name returns the name of the repository.
    Name() string
    Manifests(ctx context.Context, options ...ManifestServiceOption) (ManifestService, error)

    // Blobs returns a reference to this repository's blob service.
    Blobs(ctx context.Context) BlobStore

    // Signatures returns a reference to this repository's signatures service.
    Signatures() SignatureService
}
```
它是和repository中manifests和layer交互的函数的集合。
之后调用的manSvc.Tags函数返回在一个名字下所有tag版本。
在往后是下面的语句：
```
broadcaster, found := p.poolAdd("pull", taggedName)
broadcaster.Add(p.config.OutStream)
if found {
    return broadcaster.Wait()
}
```
poolAdd函数检测该客户端是否在对相同的一个镜像执行着pull或者push的服务，如果是就等待；不是就建立一个新的broadcaster.Buffered，该结构体保持一个对镜像的操作。
broadcaster.Add函数将新的监视OutStream加入broadcaster
函数再往下执行是一个循环
```
var layersDownloaded bool
for _, tag := range tags {
    pulledNew, err := p.pullV2Tag(broadcaster, tag, taggedName)
    if err != nil {
        return err
    }
    layersDownloaded = layersDownloaded || pulledNew
}
```
到此为止，还未真正传输过有关docker镜像的内容，只是完成了所有的配置，下面进入镜像下载环节pullV2Tag是下载镜像的最后一步，包含的内容比较多，用到的函数以及流程如下图所示：


p.repo.Manifests(context.Background())	获取manifest服务
manifest, err := manSvc.GetByTag(tag)	其中，manifests包含了可以pull、验证和运行一个镜像的所有信息。其中有内容相关的镜像id、历史、运行配置和签名。
p.validateManifest(manifest, tag)	验证manifest的有效性
p.download(d)	下载具体的image数据
p.graph.Register(d.img, reader)	在graph中注册下载的image
p.graph.SetDigest(d.img.ID, d.digest)	为image的特定layer层设定digest
p.SetDigest(p.repoInfo.LocalName, tag, firstID)	为image本身设定一个digest

分析pullRepository的整个流程之前，很有必要了解下pullRepository函数调用者的类型TagStore。TagStore是Docker镜像方面涵盖内容最多的数据结构：一方面TagStore管理Docker的Graph，另一方面TagStore还管理Docker的repository记录。除此之外，TagStore还管理着上文提到的对象pullingPool以及pushingPool，保证Docker Daemon在同一时刻，只为一个Docker Client执行同一镜像的下载或上传。TagStore结构体的定义位于docker/graph/tags.go:
```
type TagStore struct {
    path  string
    graph *Graph
    // Repositories is a map of repositories, indexed by name.
    Repositories map[string]Repository
    trustKey     libtrust.PrivateKey
    sync.Mutex
    // FIXME: move push/pull-related fields
    // to a helper type
    pullingPool     map[string]*broadcaster.Buffered
    pushingPool     map[string]*broadcaster.Buffered
    registryService *registry.Service
    eventsService   *events.Events
    trustService    *trust.Store
}
```
另一个比较重要的结构体是downloadInfo，它包含了要下载的image的配置，存放下载layer数据的临时文件的描述符，镜像的唯一digest，读layer数据的结构，镜像大小等。位于docker/graph/pull_v2.go中：
```
// downloadInfo is used to pass information from download to extractor
type downloadInfo struct {
    img         *image.Image
    tmpFile     *os.File
    digest      digest.Digest
    layer       distribution.ReadSeekCloser
    size        int64
    err         chan error
    poolKey     string
    broadcaster *broadcaster.Buffered
}
```
以下将重点分析PullV2Tag的整个流程：

go p.download(d)
```
reader := progressreader.New(progressreader.Config{
    In:        ioutil.NopCloser(io.TeeReader(layerDownload, verifier)),
    Out:       di.broadcaster,
    Formatter: p.sf,
    Size:      di.size,
    NewLines:  false,
    ID:        stringid.TruncateID(di.img.ID),
    Action:    "Downloading",
})
io.Copy(di.tmpFile, reader)
di.broadcaster.Write(p.sf.FormatProgress(stringid.TruncateID(di.img.ID), "Verifying Checksum", nil))
if !verifier.Verified() {
    err = fmt.Errorf("filesystem layer verification failed for digest %s", di.digest)
    logrus.Error(err)
    di.err <- err
    return
}
di.broadcaster.Write(p.sf.FormatProgress(stringid.TruncateID(di.img.ID), "Download complete", nil))
```
这段代码，首先为读取image的一层layer做好配置，最后的di.broadcaster.Write函数，完成layer层数据的下载。

err = p.graph.Register(d.img, reader)
```
// Apply the diff/layer
if err := graph.storeImage(img, layerData, tmp); err != nil {
    return err
}
// Commit
if err := os.Rename(tmp, graph.imageRoot(img.ID)); err != nil {
    return err
}
graph.idIndex.Add(img.ID)
}
```
storeImage将下载的特定的image的layer数据存储到驱动，这个文件会改到特定的目录，并会在graph中注册。

p.graph.SetDigest(d.img.ID, d.digest)
```
root := graph.imageRoot(id)
if err := ioutil.WriteFile(filepath.Join(root, digestFileName), []byte(dgst.String()), 0600); err != nil {
    return fmt.Errorf("Error storing digest in %s/%s: %s", root, digestFileName, err)
}
```
该函数在docker/graph/graph.go中，是Graph结构的一个函数变量，为image特定的层设定digest

p.graph.SetDigest(d.img.ID, d.digest)
```
verified, err = p.validateManifest(manifest, tag)
if err != nil {
    return false, err
}repoName = registry.NormalizeLocalName(repoName)
repoRefs, exists := store.Repositories[repoName]
if !exists {
    repoRefs = Repository{}
    store.Repositories[repoName] = repoRefs
} else if oldID, exists := repoRefs[digest]; exists && oldID != img.ID {
    return fmt.Errorf("Conflict: Digest %s is already set to image %s", digest, oldID)
}
repoRefs[digest] = img.ID
```
该函数位于docker/graph/tags.go中，是TagStore结构的一个函数成员变量，实现的功能是为image本身产生一个digest。另外，该函数也在repository中做了注册，至此pull操作完成。