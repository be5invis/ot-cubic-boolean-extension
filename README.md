# OpenType Extension: Cubic `glyf` and Booleans

## `glyf` Extension —— Cubic Curves

We activate Bits `6` and `7` in Simple Glyph Flags [1], to support new point types. In a font, if any glyph used this extension, the 0th bit of `head`.`glyphDataFormat` should be set.

- Bit `0` is set —— On-curve (`Z`)
- Bit `0` is clear:
  - Bit `7` is clear —— Quadratic off-curve (`Q`)
    - Bit `6` could have other meanings if bit `7` is clear.
  - Bit `7` is set:
    - Bit `6` is clear —— Cubic off-curve, 1st control point (`C`).
    - Bit `6` is set —— Cubic off-curve, 2nd control point (`D`).

Point auto-insertion:

| First \\ Second |     Z      |             Q              |             C              |             D              |
| :-------------- | :--------: | :------------------------: | :------------------------: | :------------------------: |
| On (`Z`)        |    N/A     |            N/A             |            N/A             |         `C` at 1/2         |
| Quad Off (`Q`)  |    N/A     |         `Z` at 1/2         |         `Z` at 2/3         | `Z` at 1/3<br />`C` at 2/3 |
| Cubic 1st (`C`) | `D` at 1/2 | `D` at 1/3<br />`Z` at 2/3 | `D` at 1/3<br />`Z` at 2/3 |            N/A             |
| Cubic 2nd (`D`) |    N/A     |         `Z` at 1/3         |         `Z` at 1/2         | `Z` at 1/3<br />`C` at 2/3 |

## Boolean Composite Extension

This extension enables complex glyphs combined together with Boolean operators. It would be extremely useful when creating variable glyphs with complex topology.

A Boolean Composer combines two layers (Background `b`, and Argument `a`) into one, and decide whether an area should be inked by the inked-ness in the two given layers. Typically a composer could be described with three binary bits `(iba)`. The meaning of each byte is:

- Bit `i`: Whether the areas should be inked, if it is inked in both the Background and Argument.
- Bit `b`: Whether the areas should be inked, if it is inked in the Background **but not** the Argument.
- Bit `a`: Whether the areas should be inked, if it is inked in the Argument **but bot** the Background.

There are 8 possible combinations:

- `(000)`: It would result an empty result, so it would be invalid or assigned to special meanings.
- `(001)`: Argument minus Background.
- `(010)`: Background minus Argument.
- `(011)`: Background XOR Argument.
- `(100)`: Intersection.
- `(101)`: Copy Argument.
- `(110)`: Copy Background.
- `(111)`: Union

### `glyf` Extension —— Boolean Composites

This extension activates three unused bits in Composite Glyph Flags [2] to support Boolean Composite Glyphs in `glyf` table. In a font, if any glyph used this extension, the 1st bit of `head`.`glyphDataFormat` should be set.

We define a new kind of composite glyph, Boolean Composite Glyph (BCG), as a glyph composed with components using Boolean expression. The existing kind of composite glyph would be named as Direct Composite (DCG).

BCGs' components could be either BCG, DCG or Simple, while for DCGs, it could only include DCGs or Simples.

Bit `15`-`13` are the Boolean Composer bits. Whether a glyph should be BCG or DCG is decided by these bits:

- If all components' Bit `15`-`13` are all clear, then it is a DCG.
  - A rasterizer should check whether its components are all DCG or Simple.
- If all components' Bit `15`-`13` are all valid Boolean Composers, then it is a BCG.
- Otherwise the glyph data is corrupted. A proper rasterizer should reject it.

For a BCG, the formula of combining components would like this:

```
(((∅ <op1> Component1) <op2> Component2) <op3> Component3) ...
```

We initialize a *background* to empty, and combine each component as the Argument in order, where the operator is defined by the Boolean Composer bits. It is recommended that for the first component in each BCG, the Composer should be `(111)` (Union).

The Boolean composition does *not* affect the coordinate computing process, also the `gvar` application and hinting. After the outline is hinted, the formula would be used in the last rasterization step to produce the final glyph.

Since components of a BCG could be another BCG, it is possible to define arbitrary-complex Boolean Composition formulas (in a tree form) that form a glyph.

Example: for a inverted glyph like `❸`, it could be defined in this manner:

- Component 1, being the circle background, using Composer `(111)` (Union).
- Component 2, being the hole, using Composer `(010)` (Background minus Argument).

### `CFF2` Extension —— Boolean Composites

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
  - The composer# is a three-bit integer defines a Boolean Composer. See the definition above.
  - Argument stack would be cleared.
  - Two layers from the Layer Stack would be popped and perform a Boolean Composition.
    - The first layer popped would be Argument.
    - The second would be Background.
    - The result would be pushed into the Layer Stack.

When the CharString if fully parsed, all the layers in the Layer Stack, *and* the active constructed path would be combined together with Union.

------

[1]: https://docs.microsoft.com/en-us/typography/opentype/spec/glyf#simple-glyph-description
[2]: https://docs.microsoft.com/en-us/typography/opentype/spec/glyf#composite-glyph-description