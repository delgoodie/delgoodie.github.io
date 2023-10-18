---
sortIndex: 1
sidebar: ue4guide
---

# Content Browser Advanced Search Syntax

<https://docs.unrealengine.com/en-US/Engine/Content/Browser/AdvancedSearchSyntax/index.html>

## Examples

- `ParentClass=='foo'` which is often what you want (so it doesn't matter how nested the BP is).  You can see what tags are available in column/detail view (when filtered to a specific type) or in the tooltip of an asset

- `NativeParentClass=`

- `Triangles >=10500 && Type == Skeletal && CollisionPrims != 0`

## Details

# Syntax Reference

Reference: <https://drive.google.com/file/d/0B6NVuVhmRCE-VHhMQ1dycXlBMWs/view>

The following table shows the available operators:
Operator (Type) Syntax Description Example
Equal (Binary) ==
=
:
Tests the value returned for the given key to see if
it is equal to the given value.

Name==Blast
Name=”Blast”
Name:Bla...

NotEqual (Binary) !=
!:
Tests the value returned for the given key to see if
it is not equal to the given value.

Name!=Blast
Name!:”Blast”

Less (Binary) < Tests the value returned for the given key to see if
it is smaller than the given value (numeric types
only).

Triangles<92

LessOrEqual (Binary) <=
<:

Tests the value returned for the given key to see if
it is smaller than, or equal to, the given value
(numeric types only).

Triangles<=92
Triangles<:92

Greater (Binary) > Tests the value returned for the given key to see if
it is larger than the given value (numeric types
only).

Triangles>92

GreaterOrEqual (Binary) >=
>:

Tests the value returned for the given key to see if
it is larger than, or equal to, the given value
(numeric types only).

Triangles>=92
Triangles>:92

Or (Binary) OR
||
|
Tests two operands and returns true if either
evaluate to true.

Blast OR Type:Blueprint
!Blast || Path:Testing
Name:”Blast” | Path:Testing...

And (Binary) AND
&&
&
Tests two operands and returns true if both
evaluate to true.

Blast AND Type:Blueprint
!Blast && Path:Testing
Name:”Blast” & Path:Testing...

Not (Pre-Unary) NOT
!
Tests the operand that follows it, and then returns
the inverted result.

NOT Blast
!”Blast”

TextCmpInvert (Pre-Unary) - Modifies a text operand so that it will return the
inverted result of the operation it is involved in.

-Blast
-”Blast”

TextCmpExact (Pre-Unary) + Modifies a text operand so that it will perform an

“exact” text comparison.

+Blast
+”Blast”
TextCmpAnchor (Pre-Unary) ... Modifies a text operand so that it will perform an ...ast

“ends with” text comparison. ...”ast”

TextCmpAnchor (Post-Unary) ... Modifies a text operand so that it will perform a

“starts with” text comparison.

Bla...
“Bla”...


Special Keys
Most keys that are available for searching come from the asset metadata that was extracted from the asset registry, however there are several
special keys that exist for all asset types. These special keys only support the Equal or NotEqual comparison operators.
Key Alias Description
Name Tests against the asset name.
Path Tests against the asset path.
Class Type Tests against the asset class.
Collection Tag Tests against the names of any collections that contain the asset.
