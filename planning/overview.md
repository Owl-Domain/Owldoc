# OwlDoc planning

OwlDoc should be a collection of tools (usable from a main CLI), along with an intermediate file format
that can represent the metadata about a project.

These tools should *not* be a magic solution for extracting and generating documentation websites in a set format/structure, but instead act as components that will be used as part of a build workflow.


## Intermediate file format

The intermediate file format should be quite basic at it's core, in order to make it easy to transform and annotate it (through the main CLI), and to make it easy to work around unknown features/extensions.

This will probably be an XML document (with `<owldoc>` as the root tag), as that will go nicely with html generation.

The main structure of the XML document will be made up of nested `<item>` tags like so:

```xml
<owldoc>
	<item type="branch" id="master">
		<item type="version" id="latest" name="v0.1">
			<item type="package" id="OwlDomain.OwlDoc.Format">
			</item>
		</item>
	</item>
</owldoc>
```

Each of the `<item>` tags will have the following attributes:

| name 	 | required | default        | description                                                                                                      |
|--------|----------|----------------|------------------------------------------------------------------------------------------------------------------|
| `id`   | yes      | N/a            | Used to uniquely identify an `<item>` tag within it's scope, will also be used as part of the final website URL. |
| `type` | yes      | N/a            | Used to select value groups during the generation stage.                                                         |
| `name` | no       | The `id` value | Used to provide a human-readable name for the `<item>`.                                                          |

Each `<item>` tag will also have a *"fake"* `full id` value associated with it (*"fake"* because it won't actually be a part of the XML document, but it's something that tooling should understand).
Which is just a concatenation of the parent ids (and the current `<item>`) with `/` as the separator. I.e: in the above example the full id for the package will be `master/latest/OwlDomain.OwlDoc.Format`.


### Parent order

The order of the parent items should be relaxed, meaning that no generator should require specific ordering of the items, meaning that all of the following examples are considered valid:

```xml
<owldoc>
	<item type="branch" id="master">
		<item type="version" id="latest" name="v0.1">
			<item type="package" id="OwlDomain.OwlDoc.Format">
			</item>
		</item>
	</item>
</owldoc>
```

```xml
<owldoc>
	<item type="version" id="latest" name="v0.1">
		<item type="branch" id="master">
			<item type="package" id="OwlDomain.OwlDoc.Format">
			</item>
		</item>
	</item>
</owldoc>
```

```xml
<owldoc>
	<item type="package" id="OwlDomain.OwlDoc.Format">
		<item type="version" id="latest" name="v0.1">
			<item type="branch" id="master">
			</item>
		</item>
	</item>
</owldoc>
```


### Scoping

The specifics of the parent/child scoping is left up to the user, to give them the biggest flexibility regarding the scope of their documentation, i.e. some projects might want to do:

```xml
<owldoc>
	<item type="package" id="OwlDomain.OwlDoc.Format">
	</item>
</owldoc>
```

While others might want to do:

```xml
<owldoc>
	<item type="organisation" id="OwlDomain">
		<item type="project" id="OwlDoc">
			<item type="branch" id="master">
				<item type="version" id="latest" name="v0.1">
					<item type="package" id="OwlDomain.OwlDoc.Format">
					</item>
				</item>
			</item>
		</item>
	</item>
</owldoc>
```

This is because the amount of scopes/nesting required will depend entirely on how the documentation is being used (e.g. Is the generated documentation only for one package? Or is it for several packages made by several different organisations? Will it support versioning? Branching? Both? Neither?), and because it would also be impossible to try and account for all of the various types of scopes that a user might want. (This is also why the parenting order shouldn't matter).

Since the scoping is relaxed, that means that metadata extractors should use the least amount of scopes as is reasonable, meaning that a metadata extractor for a C# package should not bother trying use the `organisation`, `project`, `branch`, and `version` scopes.

Instead, those scopes should be added later on (in the build pipeline) by the user through CLI transformations.


### CLI transformations

The main `owldoc` CLI tool should support several different transformation operations to help with post-processing the intermediate file format.

The CLI will contain more commands for things like annotating the document, renaming items, e.t.c, but those will be planned in more detail in a different document.

#### Parent

The `parent` subcommand will be used to add a new parent to the specified item (specified through it's full id), or to the outer most items.

```bash
owldoc parent --type="organisation" --id="OwlDomain" --item="OwlDomain.OwlDoc.Format"
owldoc parent organisation OwlDomain OwlDomain.OwlDoc.Format
owldoc parent --type="organisation" --id="OwlDomain"
owldoc parent organisation OwlDomain
```

##### Remarks

- The name can also be optionally provided with `--name="name"`.
- If the item is manually selected (i.e. through `--item`) then the command should fail if the item is not the only `<item>` in it's scope.

##### Example

```xml
<owldoc>
	<item type="package" id="OwlDomain.OwlDoc.Format">
	</item>
</owldoc>

<!-- Will turn into -->

<owldoc>
	<item type="organisation" id="OwlDomain">
		<item type="package" id="OwlDomain.OwlDoc.Format">
		</item>
	</item>
</owldoc>
```

#### Unparent

The `unparent`subcommand will be used to remove the parent of the specified item (specified through it's full id), or from the outer most items.

```bash
owldoc unparent --item="OwlDomain.OwlDoc.Format"
owldoc unparent OwlDomain.OwlDoc.Format
owldoc unparent
```

##### Remarks

- If the item is manually selected (i.e. through `--item`) then the command should fail if the item is not the only `<item>` in it's scope.

##### Example

```xml
<owldoc>
	<item type="organisation" id="OwlDomain" name="OwlDomain">
		<item type="package" id="OwlDomain.OwlDoc.Format">
		</item>
	</item>
</owldoc>

<!-- Will turn into -->

<owldoc>
	<item type="package" id="OwlDomain.OwlDoc.Format">
	</item>
</owldoc>
```

#### Reparent

The `reparent` subcommand will be used to replace the parent of the specified item (specified through it's full id), or from the outer most items.

```bash
owldoc reparent --type="organisation" --id="OwlDomain" --item="OwlDomain.OwlDoc.Format"
owldoc reparent organisation OwlDomain OwlDomain.OwlDoc.Format
owldoc reparent --type="organisation" --id="OwlDomain"
owldoc reparent organisation OwlDomain
```

##### Remarks

- The name can also be optionally provided with `--name="name"`.
- If the item is manually selected (i.e. through `--item`) then the command should fail if the item is not the only `<item>` in it's scope.

##### Example

```xml
<owldoc>
	<item type="organisation" id="foo" name="bar">
		<item type="package" id="OwlDomain.OwlDoc.Format">
		</item>
	</item>
</owldoc>

<!-- Will turn into -->

<owldoc>
	<item type="organisation" id="OwlDomain" name="OwlDomain">
		<item type="package" id="OwlDomain.OwlDoc.Format">
		</item>
	</item>
</owldoc>
```

#### Merge

The `merge` subcommand will be used to merge several intermediate files together, the purpose of this is so that each metadata extractor will extract to it's own file, which will then have to be normalised (through the parenting commands) since the scoping might be different, and then the normalised files can be merged together into one file, which is then going to be passed to the website generator.


```bash
merge output=output.xml input=file1.xml,file2.xml
merge file1.xml file2.xml output.xml
```

##### Remarks

- Content elements cannot be merged together, and if duplicates exist then the command should error.
- Merging children is done based on the id of the parent.
- The type/name attributes don't have to be included for every item in every file.
- If the type/name attributes are different for an item with the same id then the command should error.

##### Example

```xml
<owldoc>
	<item id="OwlDomain">
		<item type="package" id="OwlDomain.OwlDoc.Format">
			<item type="class" id="Foo">
			</item>
		</item>
	</item>
</owldoc>

<!-- And -->

<owldoc>
	<item type="organisation" id="OwlDomain" name="OwlDomain">
		<item id="OwlDomain.OwlDoc.Format">
			<item type="class" id="Bar">
			</item>
		</item>
	</item>
</owldoc>

<!-- Will turn into -->

<owldoc>
	<item type="organisation" id="OwlDomain" name="OwlDomain">
		<item type="package" id="OwlDomain.OwlDoc.Format">
			<item type="class" id="Bar">
			</item>
			<item type="class" id="Foo">
			</item>
		</item>
	</item>
</owldoc>
```
