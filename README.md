# Files
* buf.work.yaml - workspace root; drives work on multiple buf "modules"
* buf.yaml - module root (ie include root), specifies lint and breaking change config
* buf.gen.yaml - drives codegen (using normal protoc plugins)

# Buf Terminology
* Workspace - basically lets you list a bunch of modules and do jobs across them. Only local; BSR only sees modules, thus can't push at a workspace level, have to push per-module, and in dep graph order
  * Module - main primitive of Buf and BSR. A Module versions a set of protos and has common config for their build.
    * Full name: remote/owner/repo eg buf.build/mt-inside/greeter
    * Module's buf.yaml must contain this ^^ to be able to push/pull
* Repository - like a docker repo; multiple versions of a module

# Managed Mode
Managed mode tells buf to apply an opinionated set of language-specific options, the kind you'd set like `option go_package` in .proto files.
The idea is that these are nothing to do with the spec of the API so they shouldn't be in the API spec file.
They're config for generated clients, so buf (as protoc replacement) takes care of them.

# BSR
* Can self-host, can't find any docs
* buf push kinda just does it - no need to commit to the git repo or some buf index. Looks like it sweeps up all files with well-known names
* Can auto-gen code for you, lazily. Means that eg for go you just `go get go.buf.build/...` and there's your stubs.
  * TODO Still dunno whence go.mod - the grpc/go template only uses protoc-gen-[go,grpc] - I assume whatever's behind the GOPROXY protocol is doing it?
  * Go module version is "synthetic" - v1.{bufTemplateVesion}.{bsrModuleCommitCount} eg atm for an initial push of a module you'll get v1.2.1
  * Generation is lazy, eg for go it uses the GOPROXY protocol - `git clone go.buf.build/...` says no such repo, `go get go.buf.build/...` takes a while but produces code - go get isn't just a git clone?
  * Different build profiles are called "templates" - BSR has a few built-in, they're equiv to buf.build.yaml files, ie which protoc plugins to run and how to configure them
    * The "template in use" on BSR just seems to affect which download url it shows you - all are always available on various URLs, eg for go/grpc
      * go protos: go get go.buf.build/protocolbuffers/go/mt-inside/greeter
        * ie go.buf.build/templateOwner/templateName/moduleOwner/moduleName
      * go grpc stubs: go get go.buf.build/grpc/go/mt-inside/greeter
        * includes server stubs
  * They've got a nice approach to the usual server-side gen concerns
    * whence plugins (eg proto-gen-go) - is a repository of them, eg library/protoc-gen-go, and you can upload your own to your namespace
      * they're all packaged as docker images, which have to have the same interface as the raw plugin - listen on stdin for a binary encoding of protoc.CodeGeneratorRequest, reply on stdout with a binary protoc.CodeGeneratorResponse. Ie if it's a normal plugin, just run it
      * docker repo for these things: plugins.buf.build
      * you can even reference remote plugins in that registry to run locally
    * whither output (go just gets published to git, but eg node packages need indexing a la npm) - looks like they run a registry for each lanuage they support
      * eg go "repository": go.buf.build
    * how come it's a stand-alone module? Think they're doing it with a plugin that emits the module scaffolding
      * a "template" (generator profile) isn't like a collection of jinja, but rather a versioned collection of plugins, so I assume eg library/protoc-gen-go, buf/go-module
        * have names like "grpc/go"
  * Because of all this, and the fact they've been pushed / gone and got a load of common protos, it's super-easy to consume SDKs for things without having to do any work locally, like `docker run nixery.dev/...`, eg `go get go.buf.build/grpc/go/googleapis/googleapis/google/storage/v1` references an ambient SDK for GCP storage services


## Dependanies
Proto has no dep mgmt - you have to vendor basically. You either take a copy of the dep you want into your repo (and have to keep it up-to-date), or you "install" it, usually with an OS package that'll put the protos in /usr or something, then add that os-specific location on your protoc -I path
Buf lets your modules specify deps, which are fetched from BSR when needed, so you don't have to vendor.
Like npm etc (not quite like go), you specify the dep with a version range in your buf.yaml, then `buf mod update` to produce a lock file.
* Local module cache is TODO where?
* Seems like buf will update within that range on any operation, so if you leave a version spec off you'll always get the latest. You can stop it updating at all by pinning to one version in buf.yaml.
* Ofc you shouldn't need to cause proto files should be backwards compatible...
