## References

### Docs:
- https://substrate.dev/docs/en/knowledgebase/runtime/upgrades
- https://substrate.dev/rustdocs/v2.0.0/frame_support/traits/trait.OnRuntimeUpgrade.html

### Video
- https://www.youtube.com/watch?v=MQgDV37JrIY&t=5105s

## Some built-in functions
- on_initialize, trait: OnInitialize
- on_finalize, trait: OnFinalize
- offchain_worker
- on_runtime_upgrade, trait: OnRuntimeUpgrade

## Example

### original storages
```rust
#[derive(Encode, Decode, Clone, Debug, PartialEq)]
pub enum Version {
	V0,
	V1,
	V2,
}

impl Default for Version {
	fn default() -> Self {
		Self::V1
	}
}

#[derive(Encode, Decode, Clone, Debug, Default)]
pub struct Foo {
	a: u32
}

decl_storage! {
	trait Store for Module<T: Trait> as Migration {
		pub FooStorage: map hasher(blake2_128_concat) T::AccountId => Foo;
		pub StorageVersion: Version = Version::V0;
	}
}
```

### New storages
There're some data types added and changed.

```rust
#[derive(Encode, Decode, Clone, Debug, PartialEq)]
pub enum Status {
	Open,
	Close,
	Unknown,
}

impl Default for Status {
	fn default() -> Self {
		Self::Close
	}
}

#[derive(Encode, Decode, Clone, Debug, Default)]
pub struct Foo {
	a: u64,
	b: Status,
}

// but no storage changes.
decl_storage! {
	trait Store for Module<T: Trait> as Migration {
		pub FooStorage: map hasher(blake2_128_concat) T::AccountId => Foo;
		pub StorageVersion: Version = Version::V0;
	}
}
```

### How to migrate

Use module to include the deprecated data type
```rust
pub mod storage_migration {
	use crate::{Trait};
	use codec::{Decode, Encode};
	use frame_support::{weights::Weight, decl_module, decl_storage, IterableStorageMap, StorageValue};

	#[derive(Encode, Decode, Clone, Debug, Default)]
	pub struct Foo {
		a: u32,
	}

	decl_storage! {
		trait Store for Module<T: Trait> as Assets {
			pub FooStorage: map hasher(blake2_128_concat) T::AccountId => Foo;
		}
	}

	decl_module! {
		pub struct Module<T: Trait> for enum Call where origin: T::Origin {}
	}

	pub fn migration_to_new_foo<T: Trait>() -> Weight {
		if crate::StorageVersion::get() == crate::Version::V0 {
		    // take all old storages from previous state
			for (who, old_val) in FooStorage::<T>::drain() {
				let new_val = crate::Foo {
					a: old_val.a as u64,
					b: crate::Status::Open
				};

				crate::FooStorage::<T>::insert(who, new_val);
				crate::StorageVersion::put(crate::Version::V1);
			}

			1000
		} else {
			0
		}
	}
}

decl_module! {
    // ...
    
    fn on_runtime_upgrade() -> Weight {
    	storage_migration::migration_to_new_foo::<T>()
    }
    
    // ...
}
```

## Demonstration

Remember to update type definition to front-end.
- Old types
```
"Foo": {
    "a": "u32"
},
"Version": {
    "_enum": [
      "V0",
      "V1",
      "V2"
    ]
}
```

- New types
```
"Foo": {
    "a": "u64",
    "b": "Status"
},
"Version": {
    "_enum": [
      "V0",
      "V1",
      "V2"
    ]
},
"Status": {
    "_enum": [
      "Open",
      "Close",
      "Unknown"
    ]
}
```