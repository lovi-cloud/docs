# Use as a framework

[satelit](https://github.com/lovi-cloud/satelit) is a framework for the internal cloud. It has some components.

You can switch components to another backend that implemented `interface`.

## switch components

`NewSatelit()` ([code](https://github.com/lovi-cloud/satelit/blob/3ca3ee3840712375421a3b10059b5647e217c0aa/cmd/cmd.go#L38)) in `satelit/cmd/cmd.go` return `api.SatelitServer`.
`api.SatelitServer` has some implemented codes.

```go
    return &api.SatelitServer{
		Europa:    targetds,
		IPAM:      ipamBackend,
		Datastore: ds,
		Ganymede:  libvirtBackend,
		Scheduler: schedulerBackend,
	}, nil
```

The component has `interface` in Go. So you can switch to another implement without change the original code of satelit.

e.g.) [interface.go](https://github.com/lovi-cloud/satelit/blob/3ca3ee3840712375421a3b10059b5647e217c0aa/pkg/europa/interface.go) in europa

```go
// Europa is interface of volume operation.
type Europa interface {
	CreateVolume(ctx context.Context, name uuid.UUID, capacityGB int) (*Volume, error)
	CreateVolumeFromImage(ctx context.Context, name uuid.UUID, capacityGB int, imageID uuid.UUID) (*Volume, error)
	DeleteVolume(ctx context.Context, id string) error
	ListVolume(ctx context.Context) ([]Volume, error)
	GetVolume(ctx context.Context, id string) (*Volume, error)
	AttachVolumeTeleskop(ctx context.Context, id string, hostname string) (int, string, error)
	AttachVolumeSatelit(ctx context.Context, id string, hostname string) (int, string, error)
	DetachVolume(ctx context.Context, id string) error
	GetImage(imageID uuid.UUID) (*BaseImage, error)
	ListImage() ([]BaseImage, error)
	UploadImage(ctx context.Context, image []byte, name, description string, imageSizeGB int) (*BaseImage, error)
	DeleteImage(ctx context.Context, id uuid.UUID) error
}
```

## Components

### [Europa](https://github.com/lovi-cloud/satelit/tree/master/pkg/europa)

Europa is a component of storage.

implemented backends:

- [open-iscsi/targetd](https://github.com/lovi-cloud/satelit/tree/3ca3ee3840712375421a3b10059b5647e217c0aa/pkg/europa/targetd)
- [OceanStor Dorado](https://github.com/lovi-cloud/satelit/tree/3ca3ee3840712375421a3b10059b5647e217c0aa/pkg/europa/dorado)
- [Memory](https://github.com/lovi-cloud/satelit/tree/3ca3ee3840712375421a3b10059b5647e217c0aa/pkg/europa/memory) (for testing only. do not operation volume)

### [Ganymede](https://github.com/lovi-cloud/satelit/tree/3ca3ee3840712375421a3b10059b5647e217c0aa/pkg/ganymede)

Ganymede is a component of the virtual machine.

implemented backends:

- [libvirt](https://github.com/lovi-cloud/satelit/tree/3ca3ee3840712375421a3b10059b5647e217c0aa/pkg/ganymede/libvirt)
- [Memory](https://github.com/lovi-cloud/satelit/tree/3ca3ee3840712375421a3b10059b5647e217c0aa/pkg/ganymede/memory) (for testing only. do not operation virtual machine)

### [datastore](https://github.com/lovi-cloud/satelit/tree/3ca3ee3840712375421a3b10059b5647e217c0aa/pkg/datastore)

Datastore is a component of persistent data.

implemented backends:

- [MySQL Server](https://github.com/lovi-cloud/satelit/tree/3ca3ee3840712375421a3b10059b5647e217c0aa/pkg/datastore/mysql)
- [Memory](https://github.com/lovi-cloud/satelit/tree/3ca3ee3840712375421a3b10059b5647e217c0aa/pkg/datastore/memory) (for testing only. do not operation persistent data)

### Other Components

satelit has more components, but they depend on only `datastore`.
So you can use it directly in many cases.

- [IPam](https://github.com/lovi-cloud/satelit/tree/3ca3ee3840712375421a3b10059b5647e217c0aa/pkg/ipam): component of management network data
- [Scheduler](https://github.com/lovi-cloud/satelit/tree/3ca3ee3840712375421a3b10059b5647e217c0aa/pkg/scheduler): component of management resource in a hypervisor
