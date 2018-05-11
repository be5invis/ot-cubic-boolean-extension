# OpenType Extension: Cubic `glyf` and Booleans

## `glyf` Extension —— Cubic Cplines

We activate Bit `7` in Simple Glyph Flags [1], to support new point types. In a font, if any glyph used this extension, the 0th bit of `head`.`glyphDataFormat` should be set.

In each contour, there would be three kind of control points (“Knots”)

- Bit `0` is set —— On-curve (`Z`)
- Bit `0` is clear:
  - Bit `7` is clear —— Quadratic off-curve (`Q`)
  - Bit `7` is set —— Cubic off-curve (`B`)

Since `Z`, `B` and `Q` knots could occur in arbitrary order, a normalization pass should be preformed right after the glyph is being rasterized. It would convert all the off-curve knots to combination of on-curve points (`Z`) or cubic control points (`C`). Things between two `Z` points could only be empty of two `C` points.

The conversion is handled knot-by-knot, while the knot before and after it are also be considered to perform high-quality conversion. A curve with only `Z` and `Q` knots would be normalized just like the current TrueType specifies, and curves with only `Z` and `B` knots would be normalized as cubic B-splines.

1. Let `answer` = Empty list of (list of points).
2. For each input knot `k` at index `i`:
   1. Let `p` and `q` are the knot before and after it.
   2. If `k` is a **Z** knot:
      1. Set `answer`(`i`) to list containing 1 point:
         - **Z**(`k`)
   3. If `k` is a **B** knot:
      1. If `pkq` follows **ZBZ** pattern:
         1. Set `answer`(`i`) to a list containing 2 points:
            - **C**(mix(`p`, `k`, 2/3)) 
            - **C**(mix(`q`, `k`, 2/3))
      2. Else If `pkq` follows **ZBB** or **BBZ** pattern:
         1. Set `answer`(`i`) to a list containing 1 point:
            - **C**(`k`)
      3. Otherwise:
         1. Let `factorBefore` be 1/3 if type of `p` is **Q**, 2/3 if is **B**.
         2. Let `factorAfter` be 1/3 if type of `q` is **Q**, 2/3 if is **B**.
         3. Let `before` = mix(`p`, `k`, `factorBefore`)
         4. Let `after` = mix(`q`, `k`, `factorAfter`)
         5. Set `answer`(`i`) to a list containing 3 points: 
            - **C**(`before`)
            - **Z**(mix(`before`, `after`, 1/2))
            - **C**(`after`)
   4. If `k` is a **Q** knot:
      1. Let `halfSegBefore` = Empty point list.
      2. Let `halfSegAfter` = Empty point list.
      3. If `p` is a **Z** knot: Set `halfSegBefore` to list containing 1 point:
         - **C**(mix(`p`, `k`, 2/3))
      4. If `p` is a **Q** knot: Set `halfSegBefore` to list containing 2 points:
         - **Z**(mix(`p`, `k`, 1/2))
         - **C**(mix(`p`, `k`, 5/6))
      5. If `q` is a **Z** knot: Set `halfSegAfter` to list containing 1 point:
         - **C**(mix(`q`, `k`, 2/3))
      6. If `p` is a **Q** knot: Set `halfSegAfter` to list containing 2 points:
         - **C**(mix(`q`, `k`, 5/6))
         - **Z**(mix(`q`, `k`, 1/2))
      7. Set `answer`(`i`) to `halfSegBefore` ++ `halfSegAfter`
3. Return the `answer` flattened.

JavaScript Code

```js
function Z(pt){ return {type: "Z", x: pt.x, y: pt.y} }
function C(pt){ return {type: "C", x: pt.x, y: pt.y} }
function mix(pt1, pt2, factor) {
    return {
        x: pt1.x + (pt2.x - pt1.x) * factor,
        y: pt1.y + (pt2.y - pt1.y) * factor
    }
}
function convertKnots(knots) {
    let answer = [];
    for (let i = 0; i < knots.length; i++) {
        const k = knots[i];
        const p = knots[(i + knots.length - 1) % knots.length];
        const q = knots[(i + knots.length + 1) % knots.length];
        switch(k.type) {
            case "Z" : answer[i] = [Z(k)]; break;
            case "B" :
                if (p.type === "Z" && q.type === "Z") {
                    answer[i] = [C(mix(p, k, 2/3)), C(mix(q, k, 2/3))];
                } else if (p.type === "Z" && q.type === "B" || p.type === "B" && q.type === "Z") {
                    answer[i] = [C(k)];
                } else {
                    const factorBefore = p.type === "B" ? 2/3 : 1/3;
                    const factorAfter  = q.type === "B" ? 2/3 : 1/3;
                    const before = mix(p, k, factorBefore);
                    const after  = mix(q, k, factorAfter);
                    answer[i] = [C(before), Z(mix(before, after, 1/2)), C(after)]
                }
                break;
            case "Q":
                let halfSegBefore = [];
                let halfSegAfter = [];
                if (p.type === "Z")
                    halfSegBefore = [C(mix(p, k, 2/3))];
                if (p.type === "Q")
                    halfSegBefore = [Z(mix(p, k, 1/2)), C(mix(p, k, 5/6))];
                if (q.type === "Z")
                    halfSegAfter  = [C(mix(q, k, 2/3))];
                if (q.type === "Q")
                    halfSegAfter  = [C(mix(q, k, 5/6)), Z(mix(q, k, 1/2))];
                answer[i] = [...halfSegBefore, ...halfSegAfter]
        }
    }
    return answer.flatten()
}
```



## Boolean Composite Extension

This extension enables complex glyphs combined together with Boolean operators. It would be extremely useful when creating variable glyphs with complex topology.

A Boolean Composer combines two layers (Background `b`, and Argument `a`) into one, and decide whether an area should be inked by the inked-ness in the two given layers. Typically a composer could be described with three binary bits `(iba)`. The meaning of each byte is:

- Bit `i`: Whether the areas should be inked, if it is inked in both the Background and Argument.
- Bit `b`: Whether the areas should be inked, if it is inked in the Background **but not** the Argument.
- Bit `a`: Whether the areas should be inked, if it is inked in the Argument **but not** the Background.

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