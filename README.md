## Boxed Data - Dynamic Programming Wrapper Objects
*Note: This page uses syntax provided by the ABAP 7.4 release, for info on that check out the following page [ABAP Language News for Release 7.40](http://scn.sap.com/community/abap/blog/2013/07/22/abap-news-for-release-740). Also whilst this framework was created for 7.40, the version available is currently for 7.02. The impact of this is the move-corresponding on the boxed tableset, when used in 7.02, will explode(probably) and the to_json() method was commented out if you are using 7.40 you can comment it in*

The 'Boxed Data Framework' is a set of ABAP objects which are designed to abstract and provide usability for commonly used dynamic programming scenarios in ABAP. The framework provides a number of useful capabilities that can assist with manipulating ABAP types in a generic fashion. This document will provide a very short introduction and then immediately describe the usage.

### Introduction
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
### Usage - Elements / Basics
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
Printing the values shows the output, note that the boxed data objects have a **to_string()** method, although some output is a little more useful than others
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
### Usage - Structures
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
### Usage - Table Types
Table types can be boxed and obviously that means interacting with them will require some special functionality. The boxed type that is created when working with table types is **zcl_boxed_tableset**. Working with these objects mean that there are some relative limitations imposed by the framework. For example it is easy natively to work with ABAP table types whereas you need to work through the provided objects such as the  **zif_boxed_iterator** to iterate over a collection of boxed data. Of course the underlying representation of the contents of the boxed tableset is a generic table, which means that key access is preferred, in fact... index access is not provided out of the box for ABAP dynamic tables. It is provided by the framework, however, but it must be understood that indexed retrievals are not as efficient as key retrievals and in some cases where a hashed or sorted table has been boxed an index retrieval could return quite a confusing result.

We will continue to use the structure provided in the above structure examples in order to show the boxing of tables. The below example has a table which is already loaded with addresses.
```abap
data: ls_address type address_line,
      lt_addresses type standard table of address_line, "... Table of addresses
      lr_table type ref to zcl_boxed_tableset. "... Tableset here

"... Create some addresses (without phones)
ls_address-number = 11.
ls_address-street = 'Awesome Street'.
ls_address-state = 'SA'.
ls_address-postcode = '5000'.
append ls_address to lt_addresses.

ls_address-number = 22.
ls_address-street = 'New Thing Street'.
ls_address-state = 'NSW'.
ls_address-postcode = '1201'.
append ls_address to lt_addresses.

ls_address-number = 33.
ls_address-street = 'Other  Street'.
ls_address-state = 'ACT'.
ls_address-postcode = '2600'.
append ls_address to lt_addresses.
```
The boxed tableset objects are created the same way the structures were created in the previous section, either using the tableset object constructor or by using the **zcl_boxed_packer object**
```abap
lr_table = new #( lt_addresses ). "... Create object
lr_table ?= zcl_boxed_packer=>box( lt_addresses ). "... Use the packer (and cast)
```
Looping on a table requires the use of the iterator and its various functions
```abap
lr_table = new #( lt_addresses ). "... Create object

data(lr_iterator) = lr_table->iterator( ). "... Always creates a new iterator

while lr_iterator->has_next( ) eq abap_true.
  data(lr_line) = cast zcl_boxed_struct( lr_iterator->next( ) ). "... Requires cast

  write lr_line->get_attribute( 'street' )->to_string( ). new-line.
endwhile.
```
_**Note:** a cast is required because the next() method returns a boxed data object not a boxed structure. Of course the contents of the table does not have to be structures as line types, so the generic result is required_

An alternative is the use of the **find()** method which will find and return the first matching result. The condition must be a valid ABAP dynamic table condition, however, the ```|``` symbols may be used in place of the ```'``` symbol as it would need to be escaped (though you can use it if you wish).
```abap
data: lr_struct type ref to zcl_boxed_struct. "... Declare structure
lr_struct ?= lr_table->find( 'state eq |NSW|' ). "... Find the address in NSW
```
If more than one result is expected, the **find_all()** method can be used. It will return a **zcl_boxed_tableset** object.
```abap
data: lr_result type ref to zcl_boxed_tableset. "... The results
lr_result ?= lr_table->find_all( 'postcode cp |*00|' ). "... Find postcodes ending in 00

data(lv_str) = lr_result->to_string( ).
"> ...
"> number:11 
"> street:Awesome Street
"> state:SA
"> postcode:5000
"> phone:number:0 
"> area:
">  
"> number:33 
"> street:Other  Street
"> state:ACT
"> postcode:2600
"> phone:number:0 
"> area:
```
Of course we may need to delete entries from our table, the **remove()** method perform this function. The underlying line type is used for the deletion, you can pass the value itself or a boxed data object.
```abap
ls_address-number = 33.
ls_address-street = 'Other  Street'.
ls_address-state = 'ACT'.
ls_address-postcode = '2600'.
append ls_address to lt_addresses.

lr_table = new #( lt_addresses ). "... Create object

lr_table->remove( ls_address ). "... The last address is removed
lr_table->remove( lr_table->find( 'state = |SA|' ) ). ".. The first address is removed here
```
It would be relatively uninteresting if we couldnt find an item by index, so the following example removes index 2.
```abap
lr_table = new #( lt_addresses ). "... Create object

lr_table->remove( lr_table->find( '[2]' ) ). "... Remove index 2
```
It is possible to **insert()** a boxed data object to a tableset. The following example inserts the structure value, note that currently it either appends or inserts depending on the table type, ie hashed, sorted or standard table.
```abap
lr_table = new #( lt_addresses ). "... Create object with empty table

"... Create some addresses (without phones)
ls_address-number = 11.
ls_address-street = 'Awesome Street'.
ls_address-state = 'SA'.
ls_address-postcode = '5000'.

lr_table->insert( ls_address ). "... Insert the address structure
```
A boxed structure can be inserted with the **insert()** method, the following example is identical to the above except that it boxes the structure then inserts it. 
```abap
"... Create some addresses (without phones)
ls_address-number = 11.
ls_address-street = 'Awesome Street'.
ls_address-state = 'SA'.
ls_address-postcode = '5000'.

lr_table->insert( zcl_boxed_packer=>box( ls_address ) ). "... Insert the boxed address structure
```
Considering the examples provided, it is evident that there is a requirement to be able to get the line type of a table. The **get_line()** method will return an empty boxed data object that can be used. The below example does this, it requires a cast once again as the line type can be any type in a tableset.
```abap
data: lr_address type ref to zcl_boxed_struct, "... Now this is the structure
      lr_table type ref to zcl_boxed_tableset, "... Tableset for the addresses
      lt_addresses type standard table of address_line. "... Table type

lr_table = new #( lt_addresses ). "... Create object with empty table

lr_address ?= lr_table->get_line( ). "... Get the line out

lr_address->set_attribute( name = 'number' value = 11 ). "... Set the number
lr_address->set_attribute( name = 'street' value = 'Awesome Street' ).
lr_address->get_attribute( 'state' )->set_value( 'SA' ). "... This is equivalent to above!
lr_address->set_attribute( name = 'postcode' value = '5000' ).

lr_table->insert( lr_address ). "... Insert the address
```
Finally, in the event that the table needs to be cleared, the method **clear()** is provided.
```abap
lr_table->clear( ). "... Clears the values
```
### Usage - Path Resolution
In order to provide even greater access to complex types, and even to be able to generically access dynamic attributes within complex types like structures, the Box Path object and relative syntax was created. Using the **resolve_path()** method, as seen previously, attributes can be accessed from deeply nested structures, tables and in some cases attributes can be retrieved using the wildcard ```'*-'``` prefix.

For the purposes of the discussion the following structure will be used.
```abap
types:
  begin of phone_line,
    number type n length 7,
    area type string,
  end of phone_line,
  phone_tab type standard table of phone_line with default key,
  begin of address_line,
    number type i,
    street type string,
    state type char3,
    postcode type string,
    phone type phone_tab, "... Now this is a table
  end of address_line,
  begin of customer,
    first_name type string,
    last_name type string,
    age type i,
    primary_address type address_line,
  end of customer.
```
The structure is then populated in the following code.
```abap
data: ls_phone type phone_line,
      ls_customer type customer.

"... Setup the customer structure
ls_customer-first_name = 'Bill'.
ls_customer-last_name = 'Smith'.
ls_customer-age = 30.
ls_customer-primary_address-number = 62.
ls_customer-primary_address-postcode = '5000'.
ls_customer-primary_address-state = 'SA'.
ls_customer-primary_address-street = 'Awesome Street'.

ls_phone-area = '08'.
ls_phone-number = 8432345.
append ls_phone to ls_customer-primary_address-phone.

ls_phone-area = '08'.
ls_phone-number = 8464324.
append ls_phone to ls_customer-primary_address-phone.

ls_phone-area = '042'.
ls_phone-number = 6423423.
append ls_phone to ls_customer-primary_address-phone.
```
Data can be retrieved using the **resolve_path()** statement from nested structures. The same syntax is provided as would be seen when accessing the data in ABAP. 
```abap
data(lr_box) = zcl_boxed_packer=>box( ls_customer ). "... Box the structure

data(lr_state) = lr_box->resolve_path( 'primary_address-state' ). "... Get the state
```
Dynamic where conditions can be added to find data that is within tables. Note that the first match is returned and the search for the relevant attributes occurs via a breadth first search.
```abap
data(lr_box) = zcl_boxed_packer=>box( ls_customer ). "... Box the structure

"... Get the phone number of the first phone entry
data(lr_num) = lr_box->resolve_path( 'primary_address-phone[1]-number' ).
```
It is also possible to do a wildcard search by using the ```*-``` prefix, note that if a table is empty this would not return a value. If the result is not found the result **will be not bound**.
```abap
data(lr_box) = zcl_boxed_packer=>box( ls_customer ). "... Box the structure

"... Get the first field called 'number'
data(lr_num) = lr_box->resolve_path( '*-number' ).

write lr_num->to_string( ).
"> ...
"> 62
```
Note that the above returns the first street number! this is because during the BFS the first field found is the 'number' in the address_line structure. It is important to understand this distinction. To resolve, demonstrating the wildcard further, the path could be extended to apply a more specific relation.
```abap
data(lr_box) = zcl_boxed_packer=>box( ls_customer ). "... Box the structure

"... Get the first phone entry's number
data(lr_num) = lr_box->resolve_path( '*-phone[1]-number' ).

write lr_num->to_string( ).
"> ...
"> 8432345
```
The following is a final example.
```abap
data(lr_box) = zcl_boxed_packer=>box( ls_customer ). "... Box the structure

"... Get the first phone number where the area is '042'
data(lr_num) = lr_box->resolve_path( '*-phone[area eq |042|]-number' ).

write lr_num->to_string( ).
"> ...
"> 6423423
```
### Usage - Mappings and Bindings*
Mappings are currently implemented. Unfortunately I have no examples here to provide. Bindings were an upcomming feature that likely will not ever be complete.

### Final Notes
To install there is a .nugg file. I wont explain how to use that sorry. Yes there is a performance hit using these objects mostly due to the retrieval of the type descriptors which would be neccessary regardless. There is essentially no performance cost than doing all of the features yourself dynamically so if you are using dynamic program i can recommend you use this framework. However, the framework is definitely slower than natively using abap types so you should take this into consideration.
