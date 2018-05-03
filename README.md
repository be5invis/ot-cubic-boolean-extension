# OpenType Extension: Cubic `glyf` and Booleans

## `glyf` Extension —— Cubic Curves

We activate Bits `6` and `7` in Simple Glyph Flags [1], to support new point types. In a font, if any glyph used this extension, the 0th bit of `head`.`glyphDataFormat` should be set.

- Bit `0` is set —— On-curve (`Z`)
- Bit `0` is clear:
  - Bit `7` is clear —— Quadratic off-curve (`Q`)
    - The font parser should *always ignore* bit `6`'s value if bit `7` is clear.
  - Bit `7` is set:
    - Bit `6` is clear —— Cubic off-curve, 1st control point (`C`).
    - Bit `6` is set —— Cubic off-curve, 2nd control point (`D`).

Point auto-insertion:

| First Knot      |     Z      |             Q              |             C              |             D              |
| --------------- | :--------: | :------------------------: | :------------------------: | :------------------------: |
| On (`Z`)        |    N/A     |            N/A             |            N/A             |         `C` at 1/2         |
| Quad Off (`Q`)  |    N/A     |         `Z` at 1/2         |         `Z` at 2/3         | `Z` at 1/3<br />`C` at 2/3 |
| Cubic 1st (`C`) | `D` at 1/2 | `D` at 1/3<br />`Z` at 2/3 | `D` at 1/3<br />`Z` at 2/3 |            N/A             |
| Cubic 2nd (`D`) |    N/A     |         `Z` at 1/3         |         `Z` at 1/2         | `Z` at 1/3<br />`C` at 2/3 |

## `glyf` Extension —— Boolean Composites

This extension activates three unused bits in Composite Glyph Flags [2] to support Boolean Composite Glyphs in `glyf` table. In a font, if any glyph used this extension, the 1st bit of `head`.`glyphDataFormat` should be set.

We define a new kind of composite glyph, Boolean Composite Glyph (BCG), as a glyph composed with components using Boolean expression. The existing kind of composite glyph would be named as Direct Composite (DCG).

BCGs' components could be either BCG, DCG or Simple, while for DCGs, it could only include DCGs or Simples.

Bit `13`-`15` are used to indicate whether Boolean Composition is used in a particular glyph:

- If for a Composite Glyph, all components' Bit `13`-`15` are all clear, then it is a DCG.
  - A rasterizer should check whether its components are all DCG or Simple.
- If for a Composite Glyph, all components' Bit `13`-`15` are not all clear (at least one bit is `1` in every component), then it is a BCG.
- Otherwise the glyph data is corrupted. A proper rasterizer should reject it.

For a BCG component, the three Boolean bits define a Boolean operator that combines a virtual "drawn" glyph and the incoming component. Initially the "drawn" glyph is set to empty, and updated when a component is introduced. The formula would look like this:

```
(((∅ <op1> Component1) <op2> Component2) <op3> Component3) ...
```

The meanings of each Boolean bit are:

- Bit `13`: For the areas **inked** in the "drawn" glyph but **not inked** in the incoming component, the result should be inked when set, or not inked when clear.
- Bit `14`: For the areas **not inked** in the "drawn" glyph but **inked** in the incoming component, the result should be inked when set, or not inked when clear.
- Bit `15`: For the areas **inked** in the "drawn" glyph and **inked** in the incoming component, the result should be inked when set, or not inked when clear.

Some common operators could be defined in this manner (left to right are bit `15` to `13`):

- Union: `111`
- Intersection: `100`
- Subtraction: `001`
- Exclusive OR: `011`
- Component minus Drawn: `010`

Example: for a inverted glyph like `❸`, it could be defined in this manner:

- Component 1, being the circle background, using operator `111` (Union).
- Component 2, being the hole, using operator `001` (Subtraction).

Since components of a BCG could be another BCG, it is possible to define arbitrary-complex Boolean formulas (in a tree form) that form a glyph.

The formula would *not* affect the coordinate computing process, also the `gvar` application and hinting. After the outline is hinted, the formula would be used in the last rasterization step to produce the final glyph.

## `CFF2` Extension —— Boolean Composites

We enable two new operators and a "Layer Stack".

The CharString would be organized in this manner:

```
charString   -> initialHints? layer? contourSet?
initialHints -> hs* vs* cm* hm*
layer        -> contourSet "endlayer"
              | layer layer composer# "boolcomp"
contourSet   -> mt subpath
```

Two new operators:

- ⊢ `endlayer` (`0C 26`)
  - Argument stack must be empty.
  - Pack the constructed paths into a Layer, and push into a Layer Stack.
- composer# ⊢ `boolcomp` (`0C 27`)
  - The composer# is same as the Bits `15-13` in the `glyf` Boolean Composite Extension.
  - Argument stack would be cleared.
  - Two layers from the Layer Stack would be popped and perform a Boolean Composition. The result would be pushed into the Layer Stack.

When the CharString if fully parsed, all the layers in the Layer Stack, *and* the active constructed path would be combined together with Union.

------

[1]: https://docs.microsoft.com/en-us/typography/opentype/spec/glyf#simple-glyph-description
[2]: https://docs.microsoft.com/en-us/typography/opentype/spec/glyf#composite-glyph-description