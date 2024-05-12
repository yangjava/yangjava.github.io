---
layout: post
categories: [Docker]
description: none
keywords: Docker
---
# Docker源码image

## Image主要命令
$ docker images （所有）
$ docker images java （所有java）
$ docker images java:8 （固定tag的jave）
$ docker images --no-trunc （所有id值全长度）
$ docker images --digests （所有镜像带有digest）
$ docker images --format "{{.ID}}: {{.Repository}}" （列出两项）
$ docker images --format "table {{.ID}}\t{{.Repository}}\t{{.Tag}}" （列出三项）

## 当执行命令 docker pull，会发生哪些事情？   
docker发送image的名称+tag（或者digest）给registry服务器，服务器根据收到的image的名称+tag（或者digest），找到相应image的manifest，然后将manifest返回给docker

docker得到manifest后，读取里面image配置文件的digest(sha256)，这个sha256码就是image的ID

根据ID在本地找有没有存在同样ID的image，有的话就不用继续下载了

如果没有，那么会给registry服务器发请求（里面包含配置文件的sha256和media type），拿到image的配置文件（Image Config）

根据配置文件中的diff_ids（每个diffid对应一个layer tar包的sha256，tar包相当于layer的原始格式），在本地找对应的layer是否存在

如果layer不存在，则根据manifest里面layer的sha256和media type去服务器拿相应的layer（相当去拿压缩格式的包）。

拿到后进行解压，并检查解压后tar包的sha256能否和配置文件（Image Config）中的diff_id对的上，对不上说明有问题，下载失败

根据docker所用的后台文件系统类型，解压tar包并放到指定的目录

等所有的layer都下载完成后，整个image下载完成，就可以使用了

## Image 基础知识
镜像的子命令
```
// NewImageCommand returns a cobra command for `image` subcommands
func NewImageCommand(dockerCli *command.DockerCli) *cobra.Command {
       cmd := &cobra.Command{
              Use:   "image",
              Short: "Manage images",
              Args:  cli.NoArgs,
              RunE:  dockerCli.ShowHelp,
       }
       cmd.AddCommand(
              NewBuildCommand(dockerCli),
              NewHistoryCommand(dockerCli),
              NewImportCommand(dockerCli),
              NewLoadCommand(dockerCli),
              NewPullCommand(dockerCli),
              NewPushCommand(dockerCli),
              NewSaveCommand(dockerCli),
              NewTagCommand(dockerCli),
              newListCommand(dockerCli),
              newRemoveCommand(dockerCli),
              newInspectCommand(dockerCli),
              NewPruneCommand(dockerCli),
       )
       return cmd
}
```
拉取镜像的命令：

docker pull NAME[:TAG|@DIGEST] ，TAG为标签，DIGEST为数字摘要，也就是拉取镜像可以附带 TAG 或数字摘要等参数，或只使用镜像名（默认latest）。如果参数带 TAG 则使用 NamedTagged 描述 ，如果参数带 DIGEST 则使用Canonical 描述。

$ docker images --digests
$ docker mysql@sha256:89cc6ff6a7ac9916c3384e864fb04b8ee9415b572f872a2a4cf5b909dbbca81b
$ docker pull library/mysql

## 客户端 ImagePull    
ImagePull 函数
```
func (cli *Client) ImagePull(ctx context.Context, refStr string, options types.ImagePullOptions) (io.ReadCloser, error) {
       ref, err := reference.ParseNormalizedNamed(refStr)
 
       query := url.Values{}
       query.Set("fromImage", reference.FamiliarName(ref))
       if !options.All {
              query.Set("tag", getAPITagFromNamedRef(ref))
       }
 
       resp, err := cli.tryImageCreate(ctx, query, options.RegistryAuth)
       if resp.statusCode == http.StatusUnauthorized && options.PrivilegeFunc != nil {
              newAuthHeader, privilegeErr := options.PrivilegeFunc()
              resp, err = cli.tryImageCreate(ctx, query, newAuthHeader)
       }
      
       return resp.body, nil
}
```
ParseNormalizedNamed 中 splitDockerDomain 主要是拆分 repository 名字为 domain 和 remote 名字，没有合法的 domain 指定则使用默认的 docker.io
```
func ParseNormalizedNamed(s string) (Named, error) {
       domain, remainder := splitDockerDomain(s)
       var remoteName string
       if tagSep := strings.IndexRune(remainder, ':'); tagSep > -1 {
              remoteName = remainder[:tagSep]
       } else {
              remoteName = remainder
       }
       if strings.ToLower(remoteName) != remoteName {
              return nil, errors.New("invalid reference format: repository name must be lowercase")
       }
 
       ref, err := Parse(domain + "/" + remainder)
       named, isNamed := ref.(Named)
       return named, nil
}
```
tryImageCreate 将请求 POST 到 daemon 进程
```
func (cli *Client) tryImageCreate(ctx context.Context, query url.Values, registryAuth string) (serverResponse, error) {
       headers := map[string][]string{"X-Registry-Auth": {registryAuth}}
       return cli.post(ctx, "/images/create", query, nil, headers)
}
```

## 服务端 postImagesCreate 
postImagesCreate 解析请求参数。如果 image 存在则调用 PullImage，空则调用 ImportImage
```
func (s *imageRouter) postImagesCreate(ctx context.Context, w http.ResponseWriter, r *http.Request, vars map[string]string) error {
       var (
              image   = r.Form.Get("fromImage")
              repo    = r.Form.Get("repo")
              tag     = r.Form.Get("tag")
       )
       if image != "" { //pull
              ......
              err = s.backend.PullImage(ctx, image, tag, metaHeaders, authConfig, output)
       } else { //import
              src := r.Form.Get("fromSrc")
              err = s.backend.ImportImage(src, repo, tag, message, r.Body, output, r.Form["changes"])
       }
       
       return nil
}
```
PullImage 中 ParseNormalizedNamed 客户端分析一遍，这里在分析一遍，多多益善！主要函数 pullImageWithReference 5.1.1 节讲解
```
func (daemon *Daemon) PullImage(ctx context.Context, image, tag string, metaHeaders map[string][]string, authConfig *types.AuthConfig, outStream io.Writer) error {
       // Special case: "pull -a" may send an image name with a
       // trailing :. This is ugly, but let's not break API
       // compatibility.
       image = strings.TrimSuffix(image, ":")
 
       ref, err := reference.ParseNormalizedNamed(image)
       if err != nil {
              return err
       }
 
       if tag != "" {
              // The "tag" could actually be a digest.
              var dgst digest.Digest
              dgst, err = digest.Parse(tag)
              if err == nil {
                     ref, err = reference.WithDigest(reference.TrimNamed(ref), dgst)
              } else {
                     ref, err = reference.WithTag(ref, tag)
              }
              if err != nil {
                     return err
              }
       }
 
       return daemon.pullImageWithReference(ctx, ref, metaHeaders, authConfig, outStream)
}
```
pullImageWithReference 函数主要调用 distribution.Pull 函数，ImagePullConfig 位于 distribution/pull.go，填充一些配置信息，包括一些接口方法，结构体 5.1.1.1 所示：
```
unc (daemon *Daemon) pullImageWithReference(ctx context.Context, ref reference.Named, metaHeaders map[string][]string, authConfig *types.AuthConfig, outStream io.Writer) error 
       progressChan := make(chan progress.Progress, 100)
       writesDone := make(chan struct{})
 
       ctx, cancelFunc := context.WithCancel(ctx)
 
       go func() {
              progressutils.WriteDistributionProgress(cancelFunc, outStream, progressChan)
              close(writesDone)
       }()
 
       imagePullConfig := &distribution.ImagePullConfig{
              Config: distribution.Config{
                     MetaHeaders:      metaHeaders,
                     AuthConfig:       authConfig,
                     ProgressOutput:   progress.ChanOutput(progressChan),
                     RegistryService:  daemon.RegistryService,
                     ImageEventLogger: daemon.LogImageEvent,
                     MetadataStore:    daemon.distributionMetadataStore,
                     ImageStore:       distribution.NewImageConfigStoreFromStore(daemon.imageStore),
                     ReferenceStore:   daemon.referenceStore,
              },
              DownloadManager: daemon.downloadManager,
              Schema2Types:    distribution.ImageTypes,
       }
 
       err := distribution.Pull(ctx, ref, imagePullConfig)
       close(progressChan)
       <-writesDone
       return err
}
```
ImagePullConfig 含有 pull 配置
```
// ImagePullConfig stores pull configuration.
type ImagePullConfig struct {
       Config
 
       // DownloadManager manages concurrent pulls.
       DownloadManager RootFSDownloadManager
       // Schema2Types is the valid schema2 configuration types allowed
       // by the pull operation.
       Schema2Types []string
}
```

## 服务端 distribution Pull
Pull 函数中，ResolveRepository 函数 6.1.1 节讲解，LookupPullEndpoints 函数可以使用 /etc/docker/daemon.json 中的 endpoint，还有默认的 https://registry-1.docker.io；Pull 使用 v2 版本，路径为 distribution/pull_v2.go 中
```
func Pull(ctx context.Context, ref reference.Named, imagePullConfig *ImagePullConfig) error {
       // Resolve the Repository name from fqn to RepositoryInfo
       repoInfo, err := imagePullConfig.RegistryService.ResolveRepository(ref)
       
       // makes sure name is not `scratch`
       if err := ValidateRepoName(repoInfo.Name); err != nil {
 
       endpoints, err := imagePullConfig.RegistryService.LookupPullEndpoints(reference.Domain(repoInfo.Name))
       
       for _, endpoint := range endpoints {
              puller, err := newPuller(endpoint, repoInfo, imagePullConfig)
              
              if err := puller.Pull(ctx, ref); err != nil {
              }
 
              imagePullConfig.ImageEventLogger(reference.FamiliarString(ref), reference.FamiliarName(repoInfo.Name), "pull")
              return nil
       }
       
       return TranslatePullError(lastErr, ref)
}
```
ResolveRepository 函数拆分reference.Named，配置成RepositoryInfo（描述repository）
```
// ResolveRepository splits a repository name into its components
// and configuration of the associated registry.
func (s *DefaultService) ResolveRepository(name reference.Named) (*RepositoryInfo, error) {
       s.mu.Lock()
       defer s.mu.Unlock()
       return newRepositoryInfo(s.config, name)
}
```
Repository 结构体如下所示：
```
// RepositoryInfo describes a repository
type RepositoryInfo struct {
       Name reference.Named
       // Index points to registry information
       Index *registrytypes.IndexInfo
       // Official indicates whether the repository is considered official.
       // If the registry is official, and the normalized name does not
       // contain a '/' (e.g. "foo"), then it is considered an official repo.
       Official bool
       // Class represents the class of the repository, such as "plugin"
       // or "image".
       Class string
}
```
Pull 函数 NewV2Repository
```
func (p *v2Puller) Pull(ctx context.Context, ref reference.Named) (err error) {
       // TODO(tiborvass): was ReceiveTimeout
       p.repo, p.confirmedV2, err = NewV2Repository(ctx, p.repoInfo, p.endpoint, p.config.MetaHeaders, p.config.AuthConfig, "pull")
 
       if err = p.pullV2Repository(ctx, ref); err != nil {
              if continueOnError(err) {
                     return fallbackError{
                            err:         err,
                            confirmedV2: p.confirmedV2,
                            transportOK: true,
                     }
              }
       }
       return err
}
```
NewV2Repository 函数创建一个提供身份验证的 http 传输通道，并返回 repository 接口，验证 endpoint 可以 ping 通，如下所示：客户端的client中的registry实现了该接口，文件位于vendor/github.com/docker/distribution/registry/client/repository.go
```
func NewV2Repository(ctx context.Context, repoInfo *registry.RepositoryInfo, endpoint registry.APIEndpoint, metaHeaders http.Header, authConfig *types.AuthConfig, actions ...string) (repo distribution.Repository, foundVersion bool, err error) {
       repoName := repoInfo.Name.Name()
       // If endpoint does not support CanonicalName, use the RemoteName instead
       if endpoint.TrimHostname {
              repoName = reference.Path(repoInfo.Name)
       }
 
       challengeManager, foundVersion, err := registry.PingV2Registry(endpoint.URL, authTransport)
 
       tr := transport.NewTransport(base, modifiers...)
 
       repoNameRef, err := reference.WithName(repoName)
 
       repo, err = client.NewRepository(ctx, repoNameRef, endpoint.URL.String(), tr)
   
       return
}
```
pullV2Repository 函数主要根据不是只有名字调用函数 pullV2Tag 于第七章讲解，否则全部版本下载
```
func (p *v2Puller) pullV2Repository(ctx context.Context, ref reference.Named) (err error) {
       var layersDownloaded bool
       if !reference.IsNameOnly(ref) {
              layersDownloaded, err = p.pullV2Tag(ctx, ref)
       } else {
              tags, err := p.repo.Tags(ctx).All(ctx)
              
              // The v2 registry knows about this repository, so we will not
              // allow fallback to the v1 protocol even if we encounter an
              // error later on.
              p.confirmedV2 = true
 
              for _, tag := range tags {
                     tagRef, err := reference.WithTag(ref, tag)
                     
                     pulledNew, err := p.pullV2Tag(ctx, tagRef)
                     
                     // pulledNew is true if either new layers were downloaded OR if existing images were newly tagged
                     // TODO(tiborvass): should we change the name of `layersDownload`? What about message in WriteStatus?
                     layersDownloaded = layersDownloaded || pulledNew
              }
       }
 
       writeStatus(reference.FamiliarString(ref), p.config.ProgressOutput, layersDownloaded)
 
       return nil
}
```
服务端 pullV2Tag 函数

前面都是配置以及验证工作，pullV2Tag 才是真正的最后的下载环节，涉及的内容比较多。

Manifests 返回一个 mainfests 结构体于 7.1.1 所示
```
manSvc, err := p.repo.Manifests(ctx)
if err != nil {
       return false, err
}
```
mainfests 结构体
```

type manifests struct {
       name   reference.Named
       ub     *v2.URLBuilder
       client *http.Client
       etags  map[string]string
}
```
获取首选项的mainfest服务，这一段就是根据tag或者数字摘要进行 manSvc.Get 方法，以指定的 degest 检索，并返回 distribution.Mainfest 接口
```
if tagged, isTagged := ref.(reference.NamedTagged); isTagged {
       manifest, err = manSvc.Get(ctx, "", distribution.WithTag(tagged.Tag()))
       if err != nil {
              return false, allowV1Fallback(err)
       }
       tagOrDigest = tagged.Tag()
} else if digested, isDigested := ref.(reference.Canonical); isDigested {
       manifest, err = manSvc.Get(ctx, digested.Digest())
       if err != nil {
              return false, err
       }
       tagOrDigest = digested.Digest().String()
} else {
       return false, fmt.Errorf("internal error: reference has neither a tag nor a digest: %s", reference.FamiliarString(ref))
}
```
pullV2Tag 主要调用 pullSchema2 
```
switch v := manifest.(type) {
case *schema1.SignedManifest:
       if p.config.RequireSchema2 {
              return false, fmt.Errorf("invalid manifest: not schema2")
       }
       id, manifestDigest, err = p.pullSchema1(ctx, ref, v)
       if err != nil {
              return false, err
       }
case *schema2.DeserializedManifest:
       id, manifestDigest, err = p.pullSchema2(ctx, ref, v)
       if err != nil {
              return false, err
       }
case *manifestlist.DeserializedManifestList:
       id, manifestDigest, err = p.pullManifestList(ctx, ref, v)
       if err != nil {
              return false, err
       }
default:
       return false, errors.New("unsupported manifest format")
}
```

服务端 pullSchema2 函数

schema2ManifestDigest 函数获得 digest
```
manifestDigest, err = schema2ManifestDigest(ref, mfst)
if err != nil {
       return "", "", err
}
```
Get 方法cont /var/lib/docker/image/aufs/imagedb/content/sha256/${image-id}，查询镜像配置，如果 digest 已经存在直接返回，大致如下所示
```
if _, err := p.config.ImageStore.Get(target.Digest); err == nil {
       // If the image already exists locally, no need to pull
       // anything.
       return target.Digest, manifestDigest, nil
}
```
下载配置文件并写入 /var/lib/docker/image/aufs/imagedb/content/sha256/${image-id}
```
// Pull the image config
go func() {
       configJSON, err := p.pullSchema2Config(ctx, target.Digest)
       if err != nil {
              configErrChan <- ImageConfigPullError{Err: err}
              cancel()
              return
       }
       configChan <- configJSON
}()
```
下载个层 tar 包镜像文件
```

for _, d := range mfst.Layers {
       layerDescriptor := &v2LayerDescriptor{
              digest:            d.Digest,
              repo:              p.repo,
              repoInfo:          p.repoInfo,
              V2MetadataService: p.V2MetadataService,
              src:               d,
       }
 
       descriptors = append(descriptors, layerDescriptor)
}

```

```
if p.config.DownloadManager != nil {
       go func() {
              var (
                     err    error
                     rootFS image.RootFS
              )
              downloadRootFS := *image.NewRootFS()
              rootFS, release, err = p.config.DownloadManager.Download(ctx, downloadRootFS, descriptors, p.config.ProgressOutput)
 
              downloadedRootFS = &rootFS
              close(downloadsDone)
       }()
} 
```
将 repository 名字分析成远端域 ＋ 名字
由镜像名请求Manifest Schema v2
解析Manifest获取镜像Configuration
下载各 Layer gzip 压缩文件
验证Configuration中的RootFS.DiffIDs是否与下载（解压后）hash相同
