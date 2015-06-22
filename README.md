## Boxed Data - Dynamic Programming Wrapper Objects
*Note: This page uses syntax provided by the ABAP 7.4 release, for info on that check out the following page [ABAP Language News for Release 7.40](http://scn.sap.com/community/abap/blog/2013/07/22/abap-news-for-release-740)*

The 'Boxed Data Framework' is a set of ABAP objects which are designed to abstract and provide usability for commonly used dynamic programming scenarios in ABAP. The framework provides a number of useful capabilities that can assist with manipulating ABAP types in a generic fashion. This document will provide a very short introduction and then immediately describe the usage.

###Introduction
There are a couple of key scenarios where the framework comes in handy. To start, lets imagine the case where we want to have a structure which is stored as an object. This is a relatively common case, the structure itself is abstracted away in the object and it has 'set_attribute(name,value)' or 'get_attribute(name):value' methods. This might then form the base of another set of objects which use this object to store the data internally. Of course the underlying structure can differ per object but the access is always the same through the provided methods. I had a nice image here in a past document but as I dont have that now let me try and illustrate with an **imaginary** ABAP like language.
```abap
class Object
  private section. data: _data type ref to data. "... To contain our reference structure
  public section.  
    methods:  set_attributes( attributes ),
              set_attribute( name, value ),
              get_attributes( exporting: attributes ),
              get_attribute( ):attribute.
```
This pattern is often implemented to allow abstraction of some kind of object or 'Entity' with its own structure that can be used the same way every time. In order to eliminate the requirement to create such a structure, the BoxedStructure was created. The boxed data framework allows for abstraction of any ABAP type. For complex types, the framework provides abstraction of not only the type but also the nested access or dynamic data access in a flexible way which would otherwise be quite irritating.
```
BoxedData ~ the base abstract boxed type
            BoxedComplexType ~ abstract complex type
                        BoxedStruct ~ structures
                        BoxedTableset ~ table types
            BoxedElement ~ elements like strings, ints etc
            BoxedRef ~ data references (limited use cases)
            BoxedObject ~ object references (limited use cases)
BoxPath ~ path for accessing complex data
BoxedPacker ~ packs boxed types using the box() method
```
###Usage - Elements / Basics
The following section will just work through the usages of each boxed type. Basic elements such as strings etc will be boxed into a type of **zcl_boxed_element**.

A boxed type can be constructed as a normal object(note the use of # is abap 7.4)
```abap
data: lr_box type ref to zcl_boxed_element. "... These are 'elements'
lr_box = new #( 'This is a string'  ). "... String
lr_box = new #( 123  ). "... Int
```
If the type is not known, the **zcl_boxed_packer=>box()** method should be used
```abap
data: lr_box type ref to zcl_boxed_data. "... The base type
lr_box = zcl_boxed_packer=>box( 'This is a string' ). "... Using the packer
```
Of course the types can be boxed directly from the variable. 
```abap
data: lv_str type string value 'This is a string',
      lv_int type i value 123.

data(lr_str) = zcl_boxed_packer=>box( lv_str ). "... Use the packer!
data(lr_int) = zcl_boxed_packer=>box( lv_int ).
```
Note that this will create a copy of the data to box it which is then subsequently accessed through the reference in the object. The meaning of this is that the object will change but the originating variable will remain unchanged.
```abap
data: lv_str type string value 'This is not changed'.

data(lr_str) = zcl_boxed_packer=>box( lv_str ).
lr_str->set_value( 'but this is a new value' ).
```
Printing the values shows the output, note that the boxed data objects have a 'to_string()' method, although some output is a little more useful than others
```abap
write lv_str. new-line.
write lr_str->to_string( ).
"> ...
"> This is not changed
"> but this is a new value
```
Finally to illustrate by example, this allows types to be boxed and used in a dynamic way might otherwise be annoying.
```abap
data: lv_str type string value 'This is a string',
      lv_int type i value 123,
      lv_date type dats value '20140501',
      lv_p type p decimals 2 value '123.23',
      lt_boxed type standard table of ref to zcl_boxed_element. "... Table of elements

"... Append all the elements(usually use the boxed packer)
append new #( lv_str ) to lt_boxed.
append new #( lv_int ) to lt_boxed.
append new #( lv_date ) to lt_boxed.
append new #( lv_p ) to lt_boxed.

"... Write them out
loop at lt_boxed into data(lr_box).
  write lr_box->to_string( ). new-line.
endloop. 
"> ...
"> This is a string
"> 123
"> 20140501
"> 123.23
```
The base boxed data object provides common functionality to set and get the values. Note that the types can always be exported to their relevant type or a type of data ref can be returned. In all cases when setting a value the types must be convertable between each other. If the types dont match the **set_value()** or **get_value()** methods will always fail.
```abap
data: lv_str type string,
      lv_int type i,
      lr_data type ref to data.

data(lr_str) = zcl_boxed_packer=>box( lv_str ).

lr_str->set_value( 'This is a string' ). "... Sets a new value
lr_str->get_value( importing value = lv_str ). "... The types must match!
lr_data = lr_str->get_data( ). "... A data ref can be pulled out
*lr_str->get_value( importing value = lv_int ). "... This will fail! (the int cant be passed here!)
```
###Usage - Structures
The boxed class for working with structures is **zcl_boxed_struct**. This class is a subclass of the boxed data base class and contains all the above functionality, however, it also has some more methods specific to structures. 

For the purpose of this section the following will be the example structure. It is an address structure with a phone structure inside that.
```abap
types:
  begin of phone_line,
    number type i,
    area type string,
  end of phone_line,
  begin of address_line,
    number type i,
    street type string,
    state type char3,
    postcode type string,
    phone type phone_line,
  end of address_line.
```
Once again, the boxed type can be constructed, or it can be packed using the boxed packer. The usual case is the use of the packer as often the types are unknown since that is the real purpose of this framework, however, it is always possible to construct the object directly.
```abap
data: ls_address type address_line,
      lr_box type ref to zcl_boxed_struct. "... The structure is typed here

"... Create the address, forget the phone for now
ls_address-number = 23.
ls_address-street = 'Awesome Street'.
ls_address-state = 'SA'.
ls_address-postcode = '5000'.

"... Box this
lr_box = new #( ls_address ). "... Create struct
lr_box ?= zcl_boxed_packer=>box( ls_address ). "... Packer can / should be used
```
Notice that a cast is used when boxing the data with the packer, this is often required to allow access to the structures additional methods. Such as, the **set_attribute()** method.
```abap
"... Box this
lr_box = new #( ls_address ). "... Create struct
lr_box->set_attribute( name = 'number' value = 28 ).
```
The **get_attribute()** method returns an attribute as a boxed data type so in the example below, the **to_string()** method is available.
```abap
write lr_box->get_attribute( 'number' )->to_string( ).
"> …
"> 28
```
Both the **set_attributes()** and **get_attributes()** methods perform a move-corresponding statement.
```abap
lr_box->set_attributes( ls_address ).
lr_box->get_attributes( importing attributes = ls_address ). ".. Moves corresponding!
```
Of course it is possible to use the base boxed data **get_value()** method, which would have actually worked in the above example in place of the **get_attributes( )** method, but it is important to understand the distinction. The get_attributes method will move to a structure with corresponding field names whereas the **get_value()** method shown below requires the exact address_line type. 
```abap
lr_box->get_value( importing value = ls_address ). "... req. correct type
```
Note that we have not used the phone structure yet. Working with this structure when embedded in another structure will definitely be more painful than using the basic syntax e.g. _'ls_address-phone-number'_. But if you can imagine, if we did not know the types at compile time, this would be an unenjoyable process of _'assign component 'phone' of structure ls_address to <fs_phone>'_. Subsequently another assignment would then be needed to _'assign component 'number' of structure <fs_phone> to <fs>'_. That is the real power of the Boxed Data Framework. Lets see the comparisons together, notice that we will introduce the **resolve_path()** method here.
```abap
"... If the types are known then obviously a structure is fine to work with
ls_address-phone-number = 1234. "... Set the structure
lr_box->resolve_path( 'phone-number' )->set_value( 1234 ). "... Set the boxed value
```
When the type is known there would be no significant value, other than for generic type access, to use a boxed struct here but we see that the path can be expressed using the *zcl_box_path* syntax in the *resolve_path()* method. If we imagine that we dont know the actual type(although we do in this case) then with dynamic programming the standard syntax can be punishing _(7.4 makes it actually a little neater than before with the inline field-symbol() declaration)_.
```abap
"... When the types are unknown at compile time the dynamic syntax, is not ideal
assign component 'phone' of structure ls_address to field-symbol(<fs_phone>).
assign component 'number' of structure <fs_phone> to field-symbol(<fs_number>).
<fs_number> = 1234. "... Set the value
lr_box->resolve_path( 'phone-number' )->set_value( 1234 ). "... The boxed syntax doesnt change
```
As seen, the *resolve_path()* method can provide access to the attributes, however, note that the return of this method is a type of the abstract class *zcl_boxed_data* it is then possible to perform a cast if additional functions of the nested boxed type are required.
```abap
"... Cast into the boxed struct
data(lr_phone) = cast zcl_boxed_struct( lr_box->resolve_path( 'phone' ) ).
lr_phone->set_attribute( name = 'number' value = 1234 ). "... Set the attribute
```
###Usage - Table Types
Table types can be boxed and obviously that means interacting with them will require some special functionality. The boxed type that is created when working with table types is zcl_boxed_tableset. Working with these objects mean that there are some relative limitations imposed by the framework. For example it is easy natively to work with ABAP table types whereas you need to work through the provided objects such as the  zif_boxed_iterator to iterate over a collection of boxed data. Of course the underlying representation of the contents of the boxed tableset is a generic table, which means that key access is preferred, in fact... index access is not provided out of the box for ABAP dynamic tables. It is provided by the framework, however, but it must be understood that indexed retrievals are not as efficient as key retrievals and in some cases where a hashed or sorted table has been boxed an index retrieval could return quite a confusing result.
