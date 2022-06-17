---
title: proto是如何生成的
date: 2022-01-23 14:48:09
tags:
---

随着`grpc`协议的流行，很多时候基于`proto`生成的代码都超过写的代码了。正好得空，好好研究一下`proto`是如何生成各种代码的。

<!-- more -->


# descriptor

在`protobuf`项目下，可以找到`descriptor.proto`文件。`google`在这个文件里，定义了`proto`文件中可用的类型和选项。

每一个`proto`文件都对应一个`FileDescriptorProto`这个数据结构。

```golang
// Describes a complete .proto file.
type FileDescriptorProto struct {
	Name    *string `protobuf:"bytes,1,opt,name=name" json:"name,omitempty"`       // file name, relative to root of source tree
	Package *string `protobuf:"bytes,2,opt,name=package" json:"package,omitempty"` // e.g. "foo", "foo.bar", etc.
	Dependency []string `protobuf:"bytes,3,rep,name=dependency" json:"dependency,omitempty"`
	PublicDependency []int32 `protobuf:"varint,10,rep,name=public_dependency,json=publicDependency" json:"public_dependency,omitempty"`
	WeakDependency []int32 `protobuf:"varint,11,rep,name=weak_dependency,json=weakDependency" json:"weak_dependency,omitempty"`
	MessageType []*DescriptorProto        `protobuf:"bytes,4,rep,name=message_type,json=messageType" json:"message_type,omitempty"`
	EnumType    []*EnumDescriptorProto    `protobuf:"bytes,5,rep,name=enum_type,json=enumType" json:"enum_type,omitempty"`
	Service     []*ServiceDescriptorProto `protobuf:"bytes,6,rep,name=service" json:"service,omitempty"`
	Extension   []*FieldDescriptorProto   `protobuf:"bytes,7,rep,name=extension" json:"extension,omitempty"`
	Options     *FileOptions              `protobuf:"bytes,8,opt,name=options" json:"options,omitempty"`
	SourceCodeInfo *SourceCodeInfo `protobuf:"bytes,9,opt,name=source_code_info,json=sourceCodeInfo" json:"source_code_info,omitempty"`
	Syntax *string `protobuf:"bytes,12,opt,name=syntax" json:"syntax,omitempty"`
}
```


我们在`proto`里定义的`message`会被解析成下面这个结构。

```golang
// Describes a message type.
type DescriptorProto struct {
	Name           *string                           `protobuf:"bytes,1,opt,name=name" json:"name,omitempty"`
	Field          []*FieldDescriptorProto           `protobuf:"bytes,2,rep,name=field" json:"field,omitempty"`
	Extension      []*FieldDescriptorProto           `protobuf:"bytes,6,rep,name=extension" json:"extension,omitempty"`
	NestedType     []*DescriptorProto                `protobuf:"bytes,3,rep,name=nested_type,json=nestedType" json:"nested_type,omitempty"`
	EnumType       []*EnumDescriptorProto            `protobuf:"bytes,4,rep,name=enum_type,json=enumType" json:"enum_type,omitempty"`
	ExtensionRange []*DescriptorProto_ExtensionRange `protobuf:"bytes,5,rep,name=extension_range,json=extensionRange" json:"extension_range,omitempty"`
	OneofDecl      []*OneofDescriptorProto           `protobuf:"bytes,8,rep,name=oneof_decl,json=oneofDecl" json:"oneof_decl,omitempty"`
	Options        *MessageOptions                   `protobuf:"bytes,7,opt,name=options" json:"options,omitempty"`
	ReservedRange  []*DescriptorProto_ReservedRange  `protobuf:"bytes,9,rep,name=reserved_range,json=reservedRange" json:"reserved_range,omitempty"`
	// Reserved field names, which may not be used by fields in the same message.
	// A given name may only be reserved once.
	ReservedName []string `protobuf:"bytes,10,rep,name=reserved_name,json=reservedName" json:"reserved_name,omitempty"`
}

```

`FieldDescriptorProto`是字段字义

```golang
// Describes a field within a message.
type FieldDescriptorProto struct {
	Name   *string                     `protobuf:"bytes,1,opt,name=name" json:"name,omitempty"`
	Number *int32                      `protobuf:"varint,3,opt,name=number" json:"number,omitempty"`
	Label  *FieldDescriptorProto_Label `protobuf:"varint,4,opt,name=label,enum=google.protobuf.FieldDescriptorProto_Label" json:"label,omitempty"`
	Type *FieldDescriptorProto_Type `protobuf:"varint,5,opt,name=type,enum=google.protobuf.FieldDescriptorProto_Type" json:"type,omitempty"`
	TypeName *string `protobuf:"bytes,6,opt,name=type_name,json=typeName" json:"type_name,omitempty"`
	Extendee *string `protobuf:"bytes,2,opt,name=extendee" json:"extendee,omitempty"`
	DefaultValue *string `protobuf:"bytes,7,opt,name=default_value,json=defaultValue" json:"default_value,omitempty"`
	OneofIndex *int32 `protobuf:"varint,9,opt,name=oneof_index,json=oneofIndex" json:"oneof_index,omitempty"`
	JsonName *string       `protobuf:"bytes,10,opt,name=json_name,json=jsonName" json:"json_name,omitempty"`
	Options  *FieldOptions `protobuf:"bytes,8,opt,name=options" json:"options,omitempty"`
	Proto3Optional *bool `protobuf:"varint,17,opt,name=proto3_optional,json=proto3Optional" json:"proto3_optional,omitempty"`
}
```

`FieldDescriptorProto_Type`是一个枚举值，标记字段的类型。

```golang
type FieldDescriptorProto_Type int32

const (
	// 0 is reserved for errors.
	// Order is weird for historical reasons.
	FieldDescriptorProto_TYPE_DOUBLE FieldDescriptorProto_Type = 1
	FieldDescriptorProto_TYPE_FLOAT  FieldDescriptorProto_Type = 2
	// Not ZigZag encoded.  Negative numbers take 10 bytes.  Use TYPE_SINT64 if
	// negative values are likely.
	FieldDescriptorProto_TYPE_INT64  FieldDescriptorProto_Type = 3
	FieldDescriptorProto_TYPE_UINT64 FieldDescriptorProto_Type = 4
	// Not ZigZag encoded.  Negative numbers take 10 bytes.  Use TYPE_SINT32 if
	// negative values are likely.
	FieldDescriptorProto_TYPE_INT32   FieldDescriptorProto_Type = 5
	FieldDescriptorProto_TYPE_FIXED64 FieldDescriptorProto_Type = 6
	FieldDescriptorProto_TYPE_FIXED32 FieldDescriptorProto_Type = 7
	FieldDescriptorProto_TYPE_BOOL    FieldDescriptorProto_Type = 8
	FieldDescriptorProto_TYPE_STRING  FieldDescriptorProto_Type = 9
	// Tag-delimited aggregate.
	// Group type is deprecated and not supported in proto3. However, Proto3
	// implementations should still be able to parse the group wire format and
	// treat group fields as unknown fields.
	FieldDescriptorProto_TYPE_GROUP   FieldDescriptorProto_Type = 10
	FieldDescriptorProto_TYPE_MESSAGE FieldDescriptorProto_Type = 11 // Length-delimited aggregate.
	// New in version 2.
	FieldDescriptorProto_TYPE_BYTES    FieldDescriptorProto_Type = 12
	FieldDescriptorProto_TYPE_UINT32   FieldDescriptorProto_Type = 13
	FieldDescriptorProto_TYPE_ENUM     FieldDescriptorProto_Type = 14
	FieldDescriptorProto_TYPE_SFIXED32 FieldDescriptorProto_Type = 15
	FieldDescriptorProto_TYPE_SFIXED64 FieldDescriptorProto_Type = 16
	FieldDescriptorProto_TYPE_SINT32   FieldDescriptorProto_Type = 17 // Uses ZigZag encoding.
	FieldDescriptorProto_TYPE_SINT64   FieldDescriptorProto_Type = 18 // Uses ZigZag encoding.
)
```


`FieldDescriptorProto_Label`也是一个枚举值，`proto3`里定义数组类型时会有这个标记。

```golang
type FieldDescriptorProto_Label int32

const (
	// 0 is reserved for errors
	FieldDescriptorProto_LABEL_OPTIONAL FieldDescriptorProto_Label = 1
	FieldDescriptorProto_LABEL_REQUIRED FieldDescriptorProto_Label = 2
	FieldDescriptorProto_LABEL_REPEATED FieldDescriptorProto_Label = 3
)

```

# generator

`descriptor`决定了`proto`文件 如何被解析，`genertor`则是将解析出来的`proto`文件信息生成想要的`x.go`。


`Generator`数据结构很清晰，就是接收输入`Request`，处理成`Response`。

```golang
// Generator is the type whose methods generate the output, stored in the associated response structure.
type Generator struct {
	*bytes.Buffer

	Request  *plugin.CodeGeneratorRequest  // The input.
	Response *plugin.CodeGeneratorResponse // The output.

	Param             map[string]string // Command-line parameters.
	PackageImportPath string            // Go import path of the package we're generating code for
	ImportPrefix      string            // String to prefix to imported package file names.
	ImportMap         map[string]string // Mapping from .proto file name to import path

	Pkg map[string]string // The names under which we import support packages

	outputImportPath GoImportPath                   // Package we're generating code for.
	allFiles         []*FileDescriptor              // All files in the tree
	allFilesByName   map[string]*FileDescriptor     // All files by filename.
	genFiles         []*FileDescriptor              // Those files we will generate output for.
	file             *FileDescriptor                // The file we are compiling now.
	packageNames     map[GoImportPath]GoPackageName // Imported package names in the current file.
	usedPackages     map[GoImportPath]bool          // Packages used in current file.
	usedPackageNames map[GoPackageName]bool         // Package names used in the current file.
	addedImports     map[GoImportPath]bool          // Additional imports to emit.
	typeNameToObject map[string]Object              // Key is a fully-qualified name in input syntax.
	init             []string                       // Lines to emit in the init function.
	indent           string
	pathType         pathType // How to generate output filenames.
	writeOutput      bool
	annotateCode     bool                                       // whether to store annotations
	annotations      []*descriptor.GeneratedCodeInfo_Annotation // annotations to store
}

```

`Generator`实现的就是生成模板生成，`P`方法是打印。


```golang
// P prints the arguments to the generated output.  It handles strings and int32s, plus
// handling indirections because they may be *string, etc.  Any inputs of type AnnotatedAtoms may emit
// annotations in a .meta file in addition to outputting the atoms themselves (if g.annotateCode
// is true).
func (g *Generator) P(str ...interface{}) {
	if !g.writeOutput {
		return
	}
	g.WriteString(g.indent)
	for _, v := range str {
		switch v := v.(type) {
		case *AnnotatedAtoms:
			begin := int32(g.Len())
			for _, v := range v.atoms {
				g.printAtom(v)
			}
			if g.annotateCode {
				end := int32(g.Len())
				var path []int32
				for _, token := range strings.Split(v.path, ",") {
					val, err := strconv.ParseInt(token, 10, 32)
					if err != nil {
						g.Fail("could not parse proto AST path: ", err.Error())
					}
					path = append(path, int32(val))
				}
				g.annotations = append(g.annotations, &descriptor.GeneratedCodeInfo_Annotation{
					Path:       path,
					SourceFile: &v.source,
					Begin:      &begin,
					End:        &end,
				})
			}
		default:
			g.printAtom(v)
		}
	}
	g.WriteByte('\n')
}
```



# Plugin

```golang
// A Plugin provides functionality to add to the output during Go code generation,
// such as to produce RPC stubs.
type Plugin interface {
	// Name identifies the plugin.
	Name() string
	// Init is called once after data structures are built but before
	// code generation begins.
	Init(g *Generator)
	// Generate produces the code generated by the plugin for this file,
	// except for the imports, by calling the generator's methods P, In, and Out.
	Generate(file *FileDescriptor)
	// GenerateImports produces the import declarations for this file.
	// It is called after Generate.
	GenerateImports(file *FileDescriptor)
}
```




