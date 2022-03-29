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
* Can auto-gen code for you, lazily. Means that eg for go you just `go get go.buf.build/...` and there's your stubs. Is a bit weird
  * They have a template that includes a go.mod
  * Go module version is "synthetic" - v1.{bufTemplateVesion}.{bsrModuleCommitCount} eg atm for an initial push of a module you'll get v1.2.1
  * Lazy generate is triggered somehow - `git clone url` said no such repo, `go get url` took a while but produced code - go get isn't just a git clone?
  * Different build profiles are called "templates" - BSR has a few built-in, they're equiv to buf.build.yaml files, ie which protoc plugins to run and how to configure them
    * The "template in use" on BSR just seems to affect which download url it shows you - all are always available on various URLs, eg for go/grpc
      * go protos: go get go.buf.build/protocolbuffers/go/mt-inside/greeter
      * grpc stubs: go get go.buf.build/grpc/go/mt-inside/greeter
        * includes server stubs

## Dependanies
Proto has no dep mgmt - you have to vendor basically. You either take a copy of the dep you want into your repo (and have to keep it up-to-date), or you "install" it, usually with an OS package that'll put the protos in /usr or something, then add that os-specific location on your protoc -I path
Buf lets your modules specify deps, which are fetched from BSR when needed, so you don't have to vendor.
Like npm etc (not quite like go), you specify the dep with a version range in your buf.yaml, then `buf mod update` to produce a lock file.
* Local module cache is TODO where?
* Seems like buf will update within that range on any operation, so if you leave a version spec off you'll always get the latest. You can stop it updating at all by pinning to one version in buf.yaml.
* Ofc you shouldn't need to cause proto files should be backwards compatible...
