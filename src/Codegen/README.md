# Codegen

 This file contains the definitions of the various objects in the SMO Object Model and what properties these objects define.

## SQL Version support

Codegen.cs defines enumerations and arrays that map server version numbers to supported properties.

When SQL Server vbumps, update codegen.cs.
Lines that need attention are commented with `// VBUMP`

```C#

    private enum SingletonSupportedVersionFlags
    {
        NOT_SET = 0,
        v7_0 = 1,
        v8_0 = 2,
        v9_0 = 4,
        v10_0 = 8,
        v10_50 = 16,
        v11_0 = 32,
        v12_0 = 64,
        v13_0 = 128,
        v14_0 = 256,
        v15_0 = 512,
        v16_0 = 1024,
    }

    private static KeyValuePair<ServerVersion, int>[] m_SingletonSupportedVersion =
    {
        new KeyValuePair<ServerVersion, int>(new ServerVersion(7,0), (int)SingletonSupportedVersionFlags.v7_0),
        new KeyValuePair<ServerVersion, int>(new ServerVersion(8,0), (int)SingletonSupportedVersionFlags.v8_0),
        new KeyValuePair<ServerVersion, int>(new ServerVersion(9,0), (int)SingletonSupportedVersionFlags.v9_0),
        new KeyValuePair<ServerVersion, int>(new ServerVersion(10,0), (int)SingletonSupportedVersionFlags.v10_0),
        new KeyValuePair<ServerVersion, int>(new ServerVersion(10,50), (int)SingletonSupportedVersionFlags.v10_50),
        new KeyValuePair<ServerVersion, int>(new ServerVersion(11,0), (int)SingletonSupportedVersionFlags.v11_0),
        new KeyValuePair<ServerVersion, int>(new ServerVersion(12,0), (int)SingletonSupportedVersionFlags.v12_0),
        new KeyValuePair<ServerVersion, int>(new ServerVersion(13,0), (int)SingletonSupportedVersionFlags.v13_0),
        new KeyValuePair<ServerVersion, int>(new ServerVersion(14,0), (int)SingletonSupportedVersionFlags.v14_0),
        // The build number should probably be 65535 for all the above
        // However, that does not matter for two reasons:
        // - if there is another line after this one, we are safe: any M.m.b with b>0 observed in 
        //   in object xml files (the ones with the object definitions) will be considered
        //   "supported" by matching the next entry.
        // - we rarely seem to rely on min_build/max_build attributes.
        new KeyValuePair<ServerVersion, int>(new ServerVersion(15,0,ushort.MaxValue), (int)SingletonSupportedVersionFlags.v15_0),
        new KeyValuePair<ServerVersion, int>(new ServerVersion(16,0,ushort.MaxValue), (int)SingletonSupportedVersionFlags.v16_0),
    };

```

Similar arrays exist for cloud versions

## CFG.XML

### Property Node

Defines a property within a class definition

Below are the attributes allowed in a property element

#### Generate

The generate attribute on properties controls whether the property is automatically generated by CodeGen.exe when it's ran. If this attribute is set to false you'll need to define the property in a partial class (usually located in SMO\Main\src) that will be compiled with the generated class during the build.

#### Is_intrinsic

This controls whether the property belongs to the XSchemaProps class or the XRuntimeProps class generated inside each SMO object class. This attribute only matters if the class element has the gen_metadata attribute set to true (as this is the only time we generate the metadata classes)

#### Dmf_ignore

#### Default

The default value returned when GetPropertyDefaultValue is called (the major cases for this are when the object is in the Creating state or in Design mode).

Will also add the default value to the SfcProperty attribute, but that isn't used by SMO

#### Suppress_sfc_attribute

Will suppress writing the Sfc attributes for the properties

#### reference_type/reference_template/reference_template_parameters

These properties are used to define SfcReference attributes for this property. This is used when the property references another object but isn't actual an instance of that object (for example the property might be just a name). During serialization, SFC uses reference_template and reference_template_parameters to construct a link between the object and the referenced object. The strings in reference_template_parameters are evaluated using reflection, as properties of the current object. 

Example for Table:

```xml
    <property name="FileFormatName" generate="true" reference_type="ExternalFileFormat" reference_template="Server[@Name = '{0}']/Database[@Name = '{1}']/ExternalFileFormat[@Name='{2}']" reference_template_parameters="Parent.Parent.ConnectionContext.TrueName,Parent.Name,FileFormatName"/>
```

Generates this property:

```C#
    [SfcReference(typeof(ExternalFileFormat),"Server[@Name = '{0}']/Database[@Name = '{1}']/ExternalFileFormat[@Name='{2}']","Parent.Parent.ConnectionContext.TrueName","Parent.Name","FileFormatName")]
	[CLSCompliant(false)]
	public System.String FileFormatName
```

When the Table node is serialized, the value of the FileFormatName is stored using the evaluated template:

````xml
    <SMO:TableFileFormatName>
        <sfc:Reference sml:ref="true">
            <sml:Uri>/Server/SQLTools2019-3/Database/SfcSerialize&apos;&apos;]]]&apos;{5d00119a-0dba-4bb2-8150-9621c933a182}/ExternalFileFormat/SmoBaselineVerification__ExternalFileFormat</sml:Uri>
        </sfc:Reference>
    </SMO:TableFileFormatName>
````

### Object Node

Below are the attributes allowed in an object element

#### Class_name

The name of the class being generated

#### collection_name

The parent class's name for the object collection containing instances of this object (typically the pluralization of the class name)

#### Parent_type

The type of the Parent property. If not defined will not generate the Parent property

#### Urn

The SFC URN skeleton for the object

#### Ctor_Parent

If it exists and the value isn't  AbstractCollectionBase the "new" keyword is put in the Parent property signature (public new T Parent) as long as the value

#### New_on_get_parent

Same thing as ctor_parent, except it reads in the boolean value to decide whether to append the "new" keyword

#### Base_class

The base class for the object

#### Has_schema

If true will set the key (used for collection lookups) to a SchemaObjectKey(name, null) and will generate a constructor that takes in a schema. If false (default) the key will be SimpleObjectKey. Should be true if the object is a schema-owned object. 

#### Has_constructors

Whether the basic constructors are public (default true)

#### Implements

The list of interfaces that this class implements (comma delimited)

Some common interfaces to implement:
• IObjectPermission - if the object is a [securable](<https://docs.microsoft.com/en-us/sql/relational-databases/security/securables?view=sql-server-ver15>)

#### Sealed

Whether the class has the "sealed" keyword on it (default true)

#### Gen_body

NOTE : See src/CodeGen/gen.xml for more detailed explanation of the below

NOTE2 : If this object implements IObjectPermission this must contain "obj1,1;obj2,1;enobj,2" these are flags that tell codegen to generate the permission accessors for this object type (deny/grant/revoke methods and properties)

Uses src/CodeGen/gen.xml to insert commonly-used code snippets with minor differences

The value of this attribute is a string containing a list of sets, sets are delimited by semi-colons (;). Each set contains two values, delimited by commas (,).

e.g.

Attrib1,Body1;Attrib2,Body2…

Attributes (Attrib1/Attrib2 above) - The first value in each set is the attribute it will map to. Think of attributes as sets of declarations of variables, which are later used to fill in the Body templates. They are identified by the "id" attribute of the "attribute" element in gen.xml. These are hierarchical - an attribute can have a load_id defined which is a "base" attribute to inherit other values from

e.g. ```<attributes id='obj1' load_id='base'>``` defines an attribute with ID obj1 that also includes the sub-values from the "base" attritube

Subvalues can be one of two values :

```
a <a n='variable_name' v='variable_value' t='type_of_variable'>
defines a variable ( term used interchangeably with attribute )
alias <alias to='new_variable_name' from='old_variable_value' t='type_of_variable'>    dedefines a variable
```

Bodies (Body1/Body2 above) - The second value in each set is the body element it will map to. A body is simply a template of code to insert into the generated CS file - it allows variable substitution through the use of ```<a>``` tags (whose values are set based on the attribute part of the pair). They are identified by the "id" attribute of the "body" element in gen.xml. 

e.g. ```<body id='1' generate_outside_class='false'> … </body>``` defines a body with ID "1" that is generated within the class definition

An example of the code generated we'll use the table class as an example. In cfg.xml it includes a gen_body like this :

```gen_body="obj1,1;obj2,1;enobj,2;col1,1;col2,1;encol,2;table,server_events"```

The first set is obj1,1 - which if you look at the body element with id = 1 means that it's going to generate 8 methods :

Method name Parameters (mapped to values from attribute)
public void Deny 

• permission
• granteeNames
• columnNames

public void Deny

• permission
• granteeNames
• columnNames
• Cascade

public void Grant
• permission
• granteeNames
• columnNames

public void Grant
• permission
• granteeNames
• columnNames
• grantGrant

public void Grant
• permission
• granteeNames
• columnNames
• grantGrant
• asRole

public void Revoke
• permission
• granteeNames
• columnNames

public void Revoke
• permission
• granteeNames
• columnNames
• Cascade
• revokeGrant

public void Revoke
• permission
• granteeNames
• columnNames
• Cascade
• revokeGrant
• asRole

The parameters map to the values in the attribute obj1 as explained above. So obj1 has the following attributes :

```xml
<a n='permission' v='permission' t='ObjectPermissionSet' />
<a n='call2' v='this, ' />
```

But it also inherits the following from the "base" attribute :

```xml
<a n='call1' v='PermissionWorker.Execute(' />
<a n='granteeNames' v='granteeNames' t='System.String[]' />
<a n='cascade' v='cascade' t='bool' />
<a n='grantGrant' v='grantGrant' t='bool' />
<a n='revokeGrant' v='revokeGrant' t='bool' />
<a n='asRole' v='asRole' t='System.String' />
<a n='deny' v='PermissionState.Deny, ' />
<a n='revoke' v='PermissionState.Revoke, ' />
<a n='grant' v='PermissionState.Grant, ' />
<a n='false' v='false' />
<a n='columnNames' v='null' />
```

So you can see how the first method above (Deny with 3 parameters) uses the permission, granteeNames and columnNames values, which map to ObjectPermissionSet permission, System.String[] granteeNames and null respectively. Note that null means that parameter is ignored (so the actual method generated will only have 2 parameters)

This all ends up creating the final method signature :

```C#
public void Deny(ObjectPermissionSet permission, System.String[] granteeNames, System.String[] columnNames)
```

The rest of the method body is defined in gen.xml as 

```xml
<t v='[NL]{[NL][T]' />
		<l>
			<a n='call1' />
			<a n='deny' />
			<a n='call2' />
		</l>
		<l d=", ">
			<a n='permission' />
			<a n='granteeNames' />
			<a n='columnNames' />
		</l>
		<t v=', false, false, null' />
		<t v=');[NL]}[NL]' />
```

This will use the same process to generate the code - replacing any a/alias tags with the appropriate attribute.

The final result is the method :

```C#
public void Deny(ObjectPermissionSet permission, System.String granteeName)
{
	PermissionWorker.Execute(PermissionState.Deny, this, permission, new String [] { granteeName }, null, false, false, null);
}
```

#### Has_new

#### Singleton

#### Ini_defaults

#### Parent_has_setter

#### is_design_mode

Set this property to `true` to allow your object to participate in `DesignMode`. Such objects can be unit tested with an offline connection. Most objects should have this set.

#### Parent_mode


## Additional attributes read if cfg file isn't specified in SFC Config.xml

### Type

## Collections_codegen.proj

Collections of SMO objects are generated by the SmoCollectionGenCompile target collections_codegen.proj. This project should be built manually whenever a collection needs to be created/updated.
`msbuild collections_codegen.proj`

This takes in a template file (usually schema_generic_collection.cs for schema owned objects, or generic_collection.cs for normal collections) and replaces a set of defined tokens with the values passed in through the RemainingMacros definition.

e.g.

```xml
<SmoCollectionGenCompile Include="Database">
      <MappedTypeVariable>database</MappedTypeVariable>
      <Namespace>Microsoft.SqlServer.Management.Smo</Namespace>
      <KeyType>string</KeyType>
      <CollectionTemplate>$(SmoDirectory)\generic_collection.cs</CollectionTemplate>
      <Parent>Server</Parent>
      <RemainingMacros>/DSEALED /DDATABASE /DITEM_BY_ID</RemainingMacros>
 </SmoCollectionGenCompile>
```

MappedTypeVariable, Namespace, KeyType and Parent are all items that the target replaces appropriate macros in the template file with (MAPPED_TYPE_VAR, NAMESPACE_NAME, KEY_TYPE and PARENT respectively.

1. MappedTypeVariable - the object type name (replaces MAPPED_TYPE_VAR in the template)
2. NameSpace - Microsoft.SqlServer.Management.Smo (replaces NAMESPACE_NAME in the template)
3. KeyType - either int (For objects with an objectID but no name) or string  (for all named objects) (replaces KEY_TYPE in the template)
4. CollectionTemplate -  The template file to use
5. Parent - the parent object type (replaces PARENT in the template)
6. Remaining macros - these determine the lookup functions that will be autogenerated, typically you will include /DSEALED And /DITEM_BY_ID for all named objects, and just /DSEALED for non-named objects

You can also just create the collection manually and compile it in yourself (add it to Microsoft.SqlServer.Smo.csproj), this is useful if you're doing a lot of customization.

A mix of the two is also allowed as well. See Endpoint collection EndpointBase.cs as an example. The proj then has this definition

```xml
<SmoCollectionGenCompile Include="Endpoint">
      <MappedTypeVariable>endpoint</MappedTypeVariable>
      <Namespace>Microsoft.SqlServer.Management.Smo</Namespace>
      <KeyType>string</KeyType>
      <CollectionTemplate>$(SmoDirectory)\generic_collection.cs</CollectionTemplate>
      <Parent>Server</Parent>
      <RemainingMacros>/DSEALED /DITEM_BY_ID /DPARTIAL_KEYWORD=partial</RemainingMacros>
 </SmoCollectionGenCompile>
```

Which has the Partial keyword to allow the class definitions to be merged. 

Some classes also define their own collection templates. This can be useful if you plan on having multiple collections use the same definition. If only one collection is using a template though it's easier just to full create the collection definition and put it in the Smo folder. 

```xml
   <SmoCollectionGenCompile Include="ColumnEncryptionKeyValue">
      <MappedTypeVariable>columnEncryptionKeyValue</MappedTypeVariable>
      <Namespace>Microsoft.SqlServer.Management.Smo</Namespace>
      <KeyType>int</KeyType>
      <CollectionTemplate>$(SmoDirectory)\columnencryptionkeyvalue_generic_collection.cs</CollectionTemplate>
      <Parent>ColumnEncryptionKey</Parent>
      <RemainingMacros>/DSEALED</RemainingMacros>
    </SmoCollectionGenCompile>
```
