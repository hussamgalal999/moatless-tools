# sympy__sympy-16474

| **sympy/sympy** | `d13eb25999fb8af72c87fded526f995f5c79cb3d` |
| ---- | ---- |
| **No of patches** | 2 |
| **All found context length** | 40822 |
| **Any found context length** | 304 |
| **Avg pos** | 56.0 |
| **Min pos** | 1 |
| **Max pos** | 107 |
| **Top file pos** | 1 |
| **Missing snippets** | 6 |
| **Missing patch files** | 1 |


## Expected patch

```diff
diff --git a/sympy/printing/latex.py b/sympy/printing/latex.py
--- a/sympy/printing/latex.py
+++ b/sympy/printing/latex.py
@@ -556,20 +556,22 @@ def _print_Pow(self, expr):
                 return self._print(expr.base, exp=self._print(expr.exp))
             else:
                 tex = r"%s^{%s}"
-                exp = self._print(expr.exp)
-                # issue #12886: add parentheses around superscripts raised
-                # to powers
-                base = self.parenthesize(expr.base, PRECEDENCE['Pow'])
-                if '^' in base and expr.base.is_Symbol:
-                    base = r"\left(%s\right)" % base
-                elif (isinstance(expr.base, Derivative)
-                        and base.startswith(r'\left(')
-                        and re.match(r'\\left\(\\d?d?dot', base)
-                        and base.endswith(r'\right)')):
-                    # don't use parentheses around dotted derivative
-                    base = base[6: -7]  # remove outermost added parens
-
-                return tex % (base, exp)
+                return self._helper_print_standard_power(expr, tex)
+
+    def _helper_print_standard_power(self, expr, template):
+        exp = self._print(expr.exp)
+        # issue #12886: add parentheses around superscripts raised
+        # to powers
+        base = self.parenthesize(expr.base, PRECEDENCE['Pow'])
+        if '^' in base and expr.base.is_Symbol:
+            base = r"\left(%s\right)" % base
+        elif (isinstance(expr.base, Derivative)
+            and base.startswith(r'\left(')
+            and re.match(r'\\left\(\\d?d?dot', base)
+            and base.endswith(r'\right)')):
+            # don't use parentheses around dotted derivative
+            base = base[6: -7]  # remove outermost added parens
+        return template % (base, exp)
 
     def _print_UnevaluatedExpr(self, expr):
         return self._print(expr.args[0])
@@ -1512,7 +1514,7 @@ def _print_Transpose(self, expr):
         if not isinstance(mat, MatrixSymbol):
             return r"\left(%s\right)^{T}" % self._print(mat)
         else:
-            return "%s^{T}" % self._print(mat)
+            return "%s^{T}" % self.parenthesize(mat, precedence_traditional(expr), True)
 
     def _print_Trace(self, expr):
         mat = expr.arg
@@ -1558,28 +1560,30 @@ def _print_Mod(self, expr, exp=None):
                                  self._print(expr.args[1]))
 
     def _print_HadamardProduct(self, expr):
-        from sympy import Add, MatAdd, MatMul
+        args = expr.args
+        prec = PRECEDENCE['Pow']
+        parens = self.parenthesize
+
+        return r' \circ '.join(
+            map(lambda arg: parens(arg, prec, strict=True), args))
 
-        def parens(x):
-            if isinstance(x, (Add, MatAdd, MatMul)):
-                return r"\left(%s\right)" % self._print(x)
-            return self._print(x)
-        return r' \circ '.join(map(parens, expr.args))
+    def _print_HadamardPower(self, expr):
+        template = r"%s^{\circ {%s}}"
+        return self._helper_print_standard_power(expr, template)
 
     def _print_KroneckerProduct(self, expr):
-        from sympy import Add, MatAdd, MatMul
+        args = expr.args
+        prec = PRECEDENCE['Pow']
+        parens = self.parenthesize
 
-        def parens(x):
-            if isinstance(x, (Add, MatAdd, MatMul)):
-                return r"\left(%s\right)" % self._print(x)
-            return self._print(x)
-        return r' \otimes '.join(map(parens, expr.args))
+        return r' \otimes '.join(
+            map(lambda arg: parens(arg, prec, strict=True), args))
 
     def _print_MatPow(self, expr):
         base, exp = expr.base, expr.exp
         from sympy.matrices import MatrixSymbol
         if not isinstance(base, MatrixSymbol):
-            return r"\left(%s\right)^{%s}" % (self._print(base),
+            return "\\left(%s\\right)^{%s}" % (self._print(base),
                                               self._print(exp))
         else:
             return "%s^{%s}" % (self._print(base), self._print(exp))
diff --git a/sympy/printing/precedence.py b/sympy/printing/precedence.py
--- a/sympy/printing/precedence.py
+++ b/sympy/printing/precedence.py
@@ -24,6 +24,7 @@
 # A dictionary assigning precedence values to certain classes. These values are
 # treated like they were inherited, so not every single class has to be named
 # here.
+# Do not use this with printers other than StrPrinter
 PRECEDENCE_VALUES = {
     "Equivalent": PRECEDENCE["Xor"],
     "Xor": PRECEDENCE["Xor"],
@@ -115,8 +116,9 @@ def precedence_UnevaluatedExpr(item):
 
 
 def precedence(item):
-    """
-    Returns the precedence of a given object.
+    """Returns the precedence of a given object.
+
+    This is the precedence for StrPrinter.
     """
     if hasattr(item, "precedence"):
         return item.precedence
@@ -134,19 +136,22 @@ def precedence(item):
 
 
 def precedence_traditional(item):
-    """
-    Returns the precedence of a given object according to the traditional rules
-    of mathematics. This is the precedence for the LaTeX and pretty printer.
+    """Returns the precedence of a given object according to the
+    traditional rules of mathematics.
+
+    This is the precedence for the LaTeX and pretty printer.
     """
     # Integral, Sum, Product, Limit have the precedence of Mul in LaTeX,
     # the precedence of Atom for other printers:
-    from sympy import Integral, Sum, Product, Limit, Derivative
+    from sympy import Integral, Sum, Product, Limit, Derivative, Transpose, Adjoint
     from sympy.core.expr import UnevaluatedExpr
     from sympy.tensor.functions import TensorProduct
 
     if isinstance(item, (Integral, Sum, Product, Limit, Derivative, TensorProduct)):
         return PRECEDENCE["Mul"]
-    if (item.__class__.__name__ in ("Dot", "Cross", "Gradient", "Divergence",
+    elif isinstance(item, (Transpose, Adjoint)):
+        return PRECEDENCE["Pow"]
+    elif (item.__class__.__name__ in ("Dot", "Cross", "Gradient", "Divergence",
                                     "Curl", "Laplacian")):
         return PRECEDENCE["Mul"]-1
     elif isinstance(item, UnevaluatedExpr):

```

## Expected file changes

| File | Start line | End line | Found on position | Found file position | Context length |
| ---- | ---------- | -------- | ----------------- | ------------------- | -------------- |
| sympy/printing/latex.py | 559 | 572 | 4 | 1 | 1813
| sympy/printing/latex.py | 1515 | 1515 | 107 | 1 | 40822
| sympy/printing/latex.py | 1561 | 1582 | 1 | 1 | 304
| sympy/printing/precedence.py | 27 | 27 | - | - | -
| sympy/printing/precedence.py | 118 | 119 | - | - | -
| sympy/printing/precedence.py | 137 | 149 | - | - | -


## Problem Statement

```
Add LaTeX and pretty printers for HadamardPower
Furthermore, HadamardProduct may be extended to support the division symbol.

- [ ] Add latex printer
- [ ] Add mathml printer
- [ ] Add pretty printer

```

## Retrieved code snippets

| Position | File | Start line | End line | Tokens | Sum tokens | File tokens |
| -------- | ---- | ---------- | -------- | ------ | ---------- | ----------- |
| **-> 1 <-** | **1 sympy/printing/latex.py** | 1560 | 1591| 304 | 304 | 24863 | 
| 2 | 2 sympy/printing/pretty/pretty.py | 861 | 897| 330 | 634 | 47406 | 
| 3 | **2 sympy/printing/latex.py** | 947 | 1002| 577 | 1211 | 47406 | 
| **-> 4 <-** | **2 sympy/printing/latex.py** | 518 | 572| 602 | 1813 | 47406 | 
| 5 | 2 sympy/printing/pretty/pretty.py | 849 | 859| 124 | 1937 | 47406 | 
| 6 | **2 sympy/printing/latex.py** | 839 | 919| 767 | 2704 | 47406 | 
| 7 | **2 sympy/printing/latex.py** | 1028 | 1112| 813 | 3517 | 47406 | 
| 8 | **2 sympy/printing/latex.py** | 2222 | 2288| 734 | 4251 | 47406 | 
| 9 | **2 sympy/printing/latex.py** | 1164 | 1229| 668 | 4919 | 47406 | 
| 10 | **2 sympy/printing/latex.py** | 417 | 467| 385 | 5304 | 47406 | 
| 11 | **2 sympy/printing/latex.py** | 1 | 83| 720 | 6024 | 47406 | 
| 12 | **2 sympy/printing/latex.py** | 389 | 415| 306 | 6330 | 47406 | 
| 13 | **2 sympy/printing/latex.py** | 2109 | 2190| 791 | 7121 | 47406 | 
| 14 | 2 sympy/printing/pretty/pretty.py | 1645 | 1700| 564 | 7685 | 47406 | 
| 15 | **2 sympy/printing/latex.py** | 596 | 613| 195 | 7880 | 47406 | 
| 16 | **2 sympy/printing/latex.py** | 1881 | 1943| 542 | 8422 | 47406 | 
| 17 | 2 sympy/printing/pretty/pretty.py | 264 | 330| 583 | 9005 | 47406 | 
| 18 | **2 sympy/printing/latex.py** | 1311 | 1371| 753 | 9758 | 47406 | 
| 19 | 2 sympy/printing/pretty/pretty.py | 136 | 200| 615 | 10373 | 47406 | 
| 20 | **2 sympy/printing/latex.py** | 1957 | 2004| 457 | 10830 | 47406 | 
| 21 | 3 sympy/printing/mathml.py | 903 | 955| 472 | 11302 | 62754 | 
| 22 | 3 sympy/printing/pretty/pretty.py | 733 | 758| 284 | 11586 | 62754 | 
| 23 | 3 sympy/printing/pretty/pretty.py | 481 | 533| 428 | 12014 | 62754 | 
| 24 | **3 sympy/printing/latex.py** | 1255 | 1309| 730 | 12744 | 62754 | 
| 25 | **3 sympy/printing/latex.py** | 1754 | 1764| 141 | 12885 | 62754 | 
| 26 | **3 sympy/printing/latex.py** | 1777 | 1791| 190 | 13075 | 62754 | 
| 27 | **3 sympy/printing/latex.py** | 469 | 516| 529 | 13604 | 62754 | 
| 28 | 3 sympy/printing/mathml.py | 1776 | 1873| 808 | 14412 | 62754 | 
| 29 | **3 sympy/printing/latex.py** | 1678 | 1742| 551 | 14963 | 62754 | 
| 30 | 3 sympy/printing/mathml.py | 1128 | 1200| 595 | 15558 | 62754 | 
| 31 | 3 sympy/printing/mathml.py | 1651 | 1690| 337 | 15895 | 62754 | 
| 32 | 3 sympy/printing/mathml.py | 1401 | 1494| 838 | 16733 | 62754 | 
| 33 | 3 sympy/printing/mathml.py | 1496 | 1592| 758 | 17491 | 62754 | 
| 34 | 3 sympy/printing/mathml.py | 1336 | 1381| 397 | 17888 | 62754 | 
| 35 | **3 sympy/printing/latex.py** | 122 | 206| 693 | 18581 | 62754 | 
| 36 | 3 sympy/printing/mathml.py | 210 | 236| 226 | 18807 | 62754 | 
| 37 | 3 sympy/printing/pretty/pretty.py | 1786 | 1810| 228 | 19035 | 62754 | 
| 38 | 3 sympy/printing/mathml.py | 265 | 305| 344 | 19379 | 62754 | 
| 39 | 3 sympy/printing/mathml.py | 423 | 445| 185 | 19564 | 62754 | 
| 40 | 3 sympy/printing/pretty/pretty.py | 1376 | 1390| 194 | 19758 | 62754 | 
| 41 | 3 sympy/printing/mathml.py | 624 | 642| 160 | 19918 | 62754 | 
| 42 | **3 sympy/printing/latex.py** | 1389 | 1423| 244 | 20162 | 62754 | 
| 43 | 3 sympy/printing/mathml.py | 397 | 421| 228 | 20390 | 62754 | 
| 44 | 3 sympy/printing/pretty/pretty.py | 106 | 122| 199 | 20589 | 62754 | 
| 45 | 3 sympy/printing/pretty/pretty.py | 2266 | 2306| 403 | 20992 | 62754 | 
| 46 | 3 sympy/printing/pretty/pretty.py | 1263 | 1345| 773 | 21765 | 62754 | 
| 47 | 3 sympy/printing/mathml.py | 1264 | 1307| 383 | 22148 | 62754 | 
| 48 | 3 sympy/printing/pretty/pretty.py | 1934 | 1968| 311 | 22459 | 62754 | 
| 49 | **3 sympy/printing/latex.py** | 2006 | 2061| 414 | 22873 | 62754 | 
| 50 | **3 sympy/printing/latex.py** | 753 | 820| 656 | 23529 | 62754 | 
| 51 | **3 sympy/printing/latex.py** | 1484 | 1504| 213 | 23742 | 62754 | 
| 52 | **3 sympy/printing/latex.py** | 2088 | 2098| 124 | 23866 | 62754 | 
| 53 | 3 sympy/printing/mathml.py | 1909 | 1948| 251 | 24117 | 62754 | 
| 54 | 3 sympy/printing/pretty/pretty.py | 1748 | 1763| 184 | 24301 | 62754 | 
| 55 | 3 sympy/printing/mathml.py | 868 | 901| 286 | 24587 | 62754 | 
| 56 | 3 sympy/printing/mathml.py | 957 | 1016| 430 | 25017 | 62754 | 
| 57 | 3 sympy/printing/pretty/pretty.py | 1836 | 1889| 388 | 25405 | 62754 | 
| 58 | 3 sympy/printing/mathml.py | 1708 | 1759| 419 | 25824 | 62754 | 
| 59 | **3 sympy/printing/latex.py** | 1529 | 1548| 167 | 25991 | 62754 | 
| 60 | **3 sympy/printing/latex.py** | 1833 | 1857| 267 | 26258 | 62754 | 
| 61 | 3 sympy/printing/pretty/pretty.py | 832 | 847| 136 | 26394 | 62754 | 
| 62 | **3 sympy/printing/latex.py** | 366 | 387| 196 | 26590 | 62754 | 
| 63 | 3 sympy/printing/mathml.py | 1032 | 1069| 333 | 26923 | 62754 | 
| 64 | **3 sympy/printing/latex.py** | 326 | 346| 148 | 27071 | 62754 | 
| 65 | **3 sympy/printing/latex.py** | 645 | 676| 274 | 27345 | 62754 | 
| 66 | **3 sympy/printing/latex.py** | 1114 | 1145| 308 | 27653 | 62754 | 
| 67 | **3 sympy/printing/latex.py** | 1593 | 1643| 495 | 28148 | 62754 | 
| 68 | **3 sympy/printing/latex.py** | 1793 | 1813| 210 | 28358 | 62754 | 
| 69 | **3 sympy/printing/latex.py** | 2371 | 2542| 2411 | 30769 | 62754 | 
| 70 | 3 sympy/printing/pretty/pretty.py | 2099 | 2210| 831 | 31600 | 62754 | 
| 71 | 3 sympy/printing/mathml.py | 1692 | 1706| 140 | 31740 | 62754 | 
| 72 | 3 sympy/printing/mathml.py | 745 | 781| 380 | 32120 | 62754 | 
| 73 | 3 sympy/printing/mathml.py | 1622 | 1649| 225 | 32345 | 62754 | 
| 74 | 3 sympy/printing/mathml.py | 711 | 743| 258 | 32603 | 62754 | 
| 75 | 4 sympy/printing/str.py | 323 | 368| 366 | 32969 | 70014 | 
| 76 | **4 sympy/printing/latex.py** | 1550 | 1558| 120 | 33089 | 70014 | 
| 77 | 4 sympy/printing/pretty/pretty.py | 1215 | 1261| 403 | 33492 | 70014 | 
| 78 | **4 sympy/printing/latex.py** | 921 | 930| 136 | 33628 | 70014 | 
| 79 | **4 sympy/printing/latex.py** | 688 | 720| 325 | 33953 | 70014 | 
| 80 | 4 sympy/printing/pretty/pretty.py | 32 | 92| 535 | 34488 | 70014 | 
| 81 | 4 sympy/printing/mathml.py | 238 | 263| 211 | 34699 | 70014 | 
| 82 | **4 sympy/printing/latex.py** | 822 | 837| 162 | 34861 | 70014 | 
| 83 | **4 sympy/printing/latex.py** | 1231 | 1240| 138 | 34999 | 70014 | 
| 84 | **4 sympy/printing/latex.py** | 1242 | 1253| 187 | 35186 | 70014 | 
| 85 | 4 sympy/printing/pretty/pretty.py | 636 | 662| 261 | 35447 | 70014 | 
| 86 | 4 sympy/printing/pretty/pretty.py | 1602 | 1643| 325 | 35772 | 70014 | 
| 87 | 4 sympy/printing/pretty/pretty.py | 1555 | 1572| 173 | 35945 | 70014 | 
| 88 | 4 sympy/printing/mathml.py | 783 | 812| 239 | 36184 | 70014 | 
| 89 | **4 sympy/printing/latex.py** | 1458 | 1483| 271 | 36455 | 70014 | 
| 90 | 4 sympy/printing/mathml.py | 175 | 208| 260 | 36715 | 70014 | 
| 91 | **4 sympy/printing/latex.py** | 2100 | 2107| 118 | 36833 | 70014 | 
| 92 | 4 sympy/printing/mathml.py | 1594 | 1620| 214 | 37047 | 70014 | 
| 93 | 4 sympy/printing/mathml.py | 1202 | 1220| 176 | 37223 | 70014 | 
| 94 | 4 sympy/printing/pretty/pretty.py | 580 | 634| 471 | 37694 | 70014 | 
| 95 | **4 sympy/printing/latex.py** | 1373 | 1387| 146 | 37840 | 70014 | 
| 96 | 4 sympy/printing/mathml.py | 307 | 334| 259 | 38099 | 70014 | 
| 97 | 4 sympy/printing/pretty/pretty.py | 2388 | 2452| 612 | 38711 | 70014 | 
| 98 | 4 sympy/printing/mathml.py | 543 | 574| 302 | 39013 | 70014 | 
| 99 | **4 sympy/printing/latex.py** | 1744 | 1752| 135 | 39148 | 70014 | 
| 100 | 4 sympy/printing/mathml.py | 1761 | 1774| 130 | 39278 | 70014 | 
| 101 | 4 sympy/printing/pretty/pretty.py | 807 | 830| 216 | 39494 | 70014 | 
| 102 | 4 sympy/printing/mathml.py | 1018 | 1030| 125 | 39619 | 70014 | 
| 103 | 4 sympy/printing/mathml.py | 576 | 622| 381 | 40000 | 70014 | 
| 104 | 4 sympy/printing/mathml.py | 644 | 660| 155 | 40155 | 70014 | 
| 105 | 4 sympy/printing/mathml.py | 1875 | 1906| 277 | 40432 | 70014 | 
| 106 | 4 sympy/printing/pretty/pretty.py | 1475 | 1492| 184 | 40616 | 70014 | 
| **-> 107 <-** | **4 sympy/printing/latex.py** | 1506 | 1527| 206 | 40822 | 70014 | 
| 108 | **4 sympy/printing/latex.py** | 736 | 751| 161 | 40983 | 70014 | 
| 109 | **4 sympy/printing/latex.py** | 1766 | 1775| 134 | 41117 | 70014 | 
| 110 | **4 sympy/printing/latex.py** | 2290 | 2300| 163 | 41280 | 70014 | 
| 111 | **4 sympy/printing/latex.py** | 1425 | 1443| 135 | 41415 | 70014 | 
| 112 | **4 sympy/printing/latex.py** | 722 | 734| 159 | 41574 | 70014 | 
| 113 | 5 sympy/printing/julia.py | 214 | 256| 290 | 41864 | 75769 | 
| 114 | 5 sympy/printing/pretty/pretty.py | 1459 | 1473| 177 | 42041 | 75769 | 
| 115 | 5 sympy/printing/mathml.py | 1222 | 1261| 341 | 42382 | 75769 | 
| 116 | 5 sympy/printing/pretty/pretty.py | 226 | 241| 165 | 42547 | 75769 | 
| 117 | 5 sympy/printing/pretty/pretty.py | 1515 | 1538| 258 | 42805 | 75769 | 
| 118 | 5 sympy/printing/pretty/pretty.py | 94 | 104| 135 | 42940 | 75769 | 
| 119 | 5 sympy/printing/pretty/pretty.py | 1135 | 1189| 420 | 43360 | 75769 | 
| 120 | 5 sympy/printing/mathml.py | 1 | 16| 134 | 43494 | 75769 | 
| 121 | 5 sympy/printing/mathml.py | 689 | 709| 172 | 43666 | 75769 | 
| 122 | **5 sympy/printing/latex.py** | 1147 | 1162| 138 | 43804 | 75769 | 
| 123 | 5 sympy/printing/pretty/pretty.py | 1068 | 1104| 307 | 44111 | 75769 | 
| 124 | 5 sympy/printing/pretty/pretty.py | 535 | 578| 480 | 44591 | 75769 | 
| 125 | **5 sympy/printing/latex.py** | 932 | 945| 139 | 44730 | 75769 | 
| 126 | 5 sympy/printing/pretty/pretty.py | 2037 | 2087| 366 | 45096 | 75769 | 
| 127 | **5 sympy/printing/latex.py** | 2322 | 2332| 161 | 45257 | 75769 | 
| 128 | **5 sympy/printing/latex.py** | 1445 | 1456| 171 | 45428 | 75769 | 
| 129 | **5 sympy/printing/latex.py** | 574 | 594| 219 | 45647 | 75769 | 
| 130 | 5 sympy/printing/pretty/pretty.py | 2339 | 2371| 272 | 45919 | 75769 | 
| 131 | 5 sympy/printing/mathml.py | 814 | 866| 397 | 46316 | 75769 | 
| 132 | 5 sympy/printing/pretty/pretty.py | 1392 | 1421| 289 | 46605 | 75769 | 
| 133 | 5 sympy/printing/mathml.py | 336 | 359| 205 | 46810 | 75769 | 
| 134 | **5 sympy/printing/latex.py** | 1645 | 1676| 281 | 47091 | 75769 | 
| 135 | 5 sympy/printing/pretty/pretty.py | 124 | 134| 134 | 47225 | 75769 | 
| 136 | 5 sympy/printing/mathml.py | 361 | 395| 282 | 47507 | 75769 | 
| 137 | 5 sympy/printing/pretty/pretty.py | 1702 | 1746| 501 | 48008 | 75769 | 
| 138 | 6 sympy/interactive/printing.py | 271 | 380| 1205 | 49213 | 79760 | 
| 139 | 7 sympy/parsing/latex/_antlr/latexparser.py | 269 | 382| 947 | 50160 | 110035 | 
| 140 | 7 sympy/printing/pretty/pretty.py | 1494 | 1513| 194 | 50354 | 110035 | 
| 141 | 7 sympy/printing/mathml.py | 447 | 486| 322 | 50676 | 110035 | 
| 142 | **7 sympy/printing/latex.py** | 615 | 643| 239 | 50915 | 110035 | 
| 143 | **7 sympy/printing/latex.py** | 348 | 364| 165 | 51080 | 110035 | 
| 144 | 7 sympy/printing/mathml.py | 1071 | 1102| 259 | 51339 | 110035 | 
| 145 | **7 sympy/printing/latex.py** | 1004 | 1013| 126 | 51465 | 110035 | 
| 146 | **7 sympy/printing/latex.py** | 2312 | 2320| 129 | 51594 | 110035 | 
| 147 | 8 sympy/physics/quantum/hilbert.py | 407 | 436| 306 | 51900 | 114808 | 
| 148 | 9 sympy/printing/llvmjitcode.py | 60 | 80| 242 | 52142 | 118843 | 
| 149 | **9 sympy/printing/latex.py** | 2063 | 2073| 126 | 52268 | 118843 | 
| 150 | **9 sympy/printing/latex.py** | 2302 | 2310| 123 | 52391 | 118843 | 
| 151 | **9 sympy/printing/latex.py** | 1945 | 1955| 131 | 52522 | 118843 | 
| 152 | 9 sympy/printing/mathml.py | 1383 | 1399| 155 | 52677 | 118843 | 
| 153 | 9 sympy/printing/mathml.py | 18 | 71| 398 | 53075 | 118843 | 
| 154 | 9 sympy/printing/pretty/pretty.py | 1423 | 1442| 236 | 53311 | 118843 | 
| 155 | **9 sympy/printing/latex.py** | 2192 | 2220| 215 | 53526 | 118843 | 
| 156 | **9 sympy/printing/latex.py** | 84 | 119| 491 | 54017 | 118843 | 
| 157 | **9 sympy/printing/latex.py** | 2075 | 2086| 118 | 54135 | 118843 | 
| 158 | 9 sympy/printing/mathml.py | 662 | 687| 211 | 54346 | 118843 | 
| 159 | 9 sympy/printing/pretty/pretty.py | 332 | 367| 287 | 54633 | 118843 | 
| 160 | 9 sympy/printing/pretty/pretty.py | 2308 | 2319| 147 | 54780 | 118843 | 
| 161 | 9 sympy/printing/pretty/pretty.py | 202 | 224| 195 | 54975 | 118843 | 
| 162 | 9 sympy/printing/mathml.py | 1309 | 1334| 204 | 55179 | 118843 | 
| 163 | 9 sympy/printing/pretty/pretty.py | 999 | 1031| 354 | 55533 | 118843 | 
| 164 | 9 sympy/parsing/latex/_antlr/latexparser.py | 222 | 268| 778 | 56311 | 118843 | 
| 165 | 10 sympy/printing/mathematica.py | 1 | 36| 442 | 56753 | 120898 | 
| 166 | 10 sympy/parsing/latex/_antlr/latexparser.py | 590 | 636| 479 | 57232 | 120898 | 
| 167 | **10 sympy/printing/latex.py** | 1015 | 1026| 156 | 57388 | 120898 | 
| 168 | 10 sympy/printing/pretty/pretty.py | 2248 | 2264| 190 | 57578 | 120898 | 
| 169 | 10 sympy/printing/pretty/pretty.py | 388 | 403| 160 | 57738 | 120898 | 
| 170 | 10 sympy/printing/pretty/pretty.py | 405 | 479| 607 | 58345 | 120898 | 
| 171 | 10 sympy/printing/pretty/pretty.py | 1540 | 1553| 180 | 58525 | 120898 | 
| 172 | 10 sympy/printing/pretty/pretty.py | 2212 | 2229| 180 | 58705 | 120898 | 
| 173 | 10 sympy/printing/pretty/pretty.py | 761 | 780| 207 | 58912 | 120898 | 
| 174 | 10 sympy/printing/pretty/pretty.py | 1970 | 1993| 227 | 59139 | 120898 | 
| 175 | 11 sympy/printing/lambdarepr.py | 1 | 55| 377 | 59516 | 122017 | 
| 176 | 11 sympy/printing/pretty/pretty.py | 783 | 805| 224 | 59740 | 122017 | 
| 177 | 11 sympy/printing/pretty/pretty.py | 1574 | 1600| 249 | 59989 | 122017 | 
| 178 | 12 sympy/printing/codeprinter.py | 386 | 437| 507 | 60496 | 126437 | 
| 179 | 12 sympy/printing/mathml.py | 1104 | 1126| 182 | 60678 | 126437 | 
| 180 | 12 sympy/printing/pretty/pretty.py | 1106 | 1133| 212 | 60890 | 126437 | 
| 181 | 12 sympy/printing/llvmjitcode.py | 82 | 110| 267 | 61157 | 126437 | 
| 182 | 13 sympy/printing/pycode.py | 551 | 609| 774 | 61931 | 133039 | 
| 183 | 13 sympy/printing/mathematica.py | 212 | 239| 321 | 62252 | 133039 | 
| 184 | **13 sympy/printing/latex.py** | 1859 | 1879| 220 | 62472 | 133039 | 
| 185 | **13 sympy/printing/latex.py** | 678 | 686| 128 | 62600 | 133039 | 
| 186 | 13 sympy/printing/pretty/pretty.py | 1347 | 1374| 244 | 62844 | 133039 | 
| 187 | 13 sympy/printing/pretty/pretty.py | 2321 | 2337| 184 | 63028 | 133039 | 
| 188 | 13 sympy/printing/codeprinter.py | 439 | 488| 449 | 63477 | 133039 | 
| 189 | 13 sympy/printing/pretty/pretty.py | 978 | 997| 217 | 63694 | 133039 | 
| 190 | 13 sympy/printing/pretty/pretty.py | 2454 | 2462| 119 | 63813 | 133039 | 
| 191 | 13 sympy/printing/pretty/pretty.py | 899 | 976| 764 | 64577 | 133039 | 
| 192 | 13 sympy/physics/quantum/hilbert.py | 518 | 547| 295 | 64872 | 133039 | 
| 193 | 13 sympy/printing/mathml.py | 489 | 541| 467 | 65339 | 133039 | 
| 194 | 13 sympy/printing/pretty/pretty.py | 2464 | 2521| 449 | 65788 | 133039 | 
| 195 | **13 sympy/printing/latex.py** | 2543 | 2573| 326 | 66114 | 133039 | 
| 196 | 14 sympy/printing/octave.py | 231 | 252| 172 | 66286 | 139548 | 
| 197 | 14 sympy/printing/mathml.py | 73 | 122| 503 | 66789 | 139548 | 
| 198 | 15 sympy/printing/glsl.py | 284 | 303| 281 | 67070 | 144361 | 
| 199 | 15 sympy/printing/octave.py | 136 | 208| 700 | 67770 | 144361 | 
| 200 | 15 sympy/printing/julia.py | 119 | 191| 697 | 68467 | 144361 | 
| 201 | 16 sympy/physics/vector/printing.py | 44 | 119| 662 | 69129 | 147768 | 
| 202 | 16 sympy/printing/mathematica.py | 39 | 143| 684 | 69813 | 147768 | 
| 203 | 16 sympy/printing/pretty/pretty.py | 1191 | 1213| 180 | 69993 | 147768 | 
| 204 | 16 sympy/printing/julia.py | 194 | 211| 198 | 70191 | 147768 | 
| 205 | 16 sympy/printing/str.py | 524 | 555| 398 | 70589 | 147768 | 
| 206 | 16 sympy/printing/pretty/pretty.py | 1 | 29| 276 | 70865 | 147768 | 
| 207 | 17 sympy/printing/jscode.py | 100 | 110| 133 | 70998 | 150583 | 
| 208 | 17 sympy/printing/pretty/pretty.py | 1765 | 1784| 199 | 71197 | 150583 | 
| 209 | 17 sympy/printing/str.py | 204 | 248| 482 | 71679 | 150583 | 
| 210 | 17 sympy/printing/pretty/pretty.py | 1444 | 1457| 142 | 71821 | 150583 | 
| 211 | 17 sympy/printing/pretty/pretty.py | 2231 | 2246| 179 | 72000 | 150583 | 
| 212 | **17 sympy/printing/latex.py** | 208 | 229| 219 | 72219 | 150583 | 
| 213 | 17 sympy/printing/glsl.py | 247 | 282| 303 | 72522 | 150583 | 
| 214 | 18 sympy/printing/fcode.py | 254 | 295| 340 | 72862 | 158557 | 
| 215 | 18 sympy/printing/pretty/pretty.py | 1812 | 1834| 193 | 73055 | 158557 | 
| 216 | 18 sympy/printing/str.py | 70 | 158| 805 | 73860 | 158557 | 


## Missing Patch Files

 * 1: sympy/printing/latex.py
 * 2: sympy/printing/precedence.py

### Hint

```
See: https://github.com/sympy/sympy/pull/16443#issuecomment-476504237 and subsequent discussion.
```

## Patch

```diff
diff --git a/sympy/printing/latex.py b/sympy/printing/latex.py
--- a/sympy/printing/latex.py
+++ b/sympy/printing/latex.py
@@ -556,20 +556,22 @@ def _print_Pow(self, expr):
                 return self._print(expr.base, exp=self._print(expr.exp))
             else:
                 tex = r"%s^{%s}"
-                exp = self._print(expr.exp)
-                # issue #12886: add parentheses around superscripts raised
-                # to powers
-                base = self.parenthesize(expr.base, PRECEDENCE['Pow'])
-                if '^' in base and expr.base.is_Symbol:
-                    base = r"\left(%s\right)" % base
-                elif (isinstance(expr.base, Derivative)
-                        and base.startswith(r'\left(')
-                        and re.match(r'\\left\(\\d?d?dot', base)
-                        and base.endswith(r'\right)')):
-                    # don't use parentheses around dotted derivative
-                    base = base[6: -7]  # remove outermost added parens
-
-                return tex % (base, exp)
+                return self._helper_print_standard_power(expr, tex)
+
+    def _helper_print_standard_power(self, expr, template):
+        exp = self._print(expr.exp)
+        # issue #12886: add parentheses around superscripts raised
+        # to powers
+        base = self.parenthesize(expr.base, PRECEDENCE['Pow'])
+        if '^' in base and expr.base.is_Symbol:
+            base = r"\left(%s\right)" % base
+        elif (isinstance(expr.base, Derivative)
+            and base.startswith(r'\left(')
+            and re.match(r'\\left\(\\d?d?dot', base)
+            and base.endswith(r'\right)')):
+            # don't use parentheses around dotted derivative
+            base = base[6: -7]  # remove outermost added parens
+        return template % (base, exp)
 
     def _print_UnevaluatedExpr(self, expr):
         return self._print(expr.args[0])
@@ -1512,7 +1514,7 @@ def _print_Transpose(self, expr):
         if not isinstance(mat, MatrixSymbol):
             return r"\left(%s\right)^{T}" % self._print(mat)
         else:
-            return "%s^{T}" % self._print(mat)
+            return "%s^{T}" % self.parenthesize(mat, precedence_traditional(expr), True)
 
     def _print_Trace(self, expr):
         mat = expr.arg
@@ -1558,28 +1560,30 @@ def _print_Mod(self, expr, exp=None):
                                  self._print(expr.args[1]))
 
     def _print_HadamardProduct(self, expr):
-        from sympy import Add, MatAdd, MatMul
+        args = expr.args
+        prec = PRECEDENCE['Pow']
+        parens = self.parenthesize
+
+        return r' \circ '.join(
+            map(lambda arg: parens(arg, prec, strict=True), args))
 
-        def parens(x):
-            if isinstance(x, (Add, MatAdd, MatMul)):
-                return r"\left(%s\right)" % self._print(x)
-            return self._print(x)
-        return r' \circ '.join(map(parens, expr.args))
+    def _print_HadamardPower(self, expr):
+        template = r"%s^{\circ {%s}}"
+        return self._helper_print_standard_power(expr, template)
 
     def _print_KroneckerProduct(self, expr):
-        from sympy import Add, MatAdd, MatMul
+        args = expr.args
+        prec = PRECEDENCE['Pow']
+        parens = self.parenthesize
 
-        def parens(x):
-            if isinstance(x, (Add, MatAdd, MatMul)):
-                return r"\left(%s\right)" % self._print(x)
-            return self._print(x)
-        return r' \otimes '.join(map(parens, expr.args))
+        return r' \otimes '.join(
+            map(lambda arg: parens(arg, prec, strict=True), args))
 
     def _print_MatPow(self, expr):
         base, exp = expr.base, expr.exp
         from sympy.matrices import MatrixSymbol
         if not isinstance(base, MatrixSymbol):
-            return r"\left(%s\right)^{%s}" % (self._print(base),
+            return "\\left(%s\\right)^{%s}" % (self._print(base),
                                               self._print(exp))
         else:
             return "%s^{%s}" % (self._print(base), self._print(exp))
diff --git a/sympy/printing/precedence.py b/sympy/printing/precedence.py
--- a/sympy/printing/precedence.py
+++ b/sympy/printing/precedence.py
@@ -24,6 +24,7 @@
 # A dictionary assigning precedence values to certain classes. These values are
 # treated like they were inherited, so not every single class has to be named
 # here.
+# Do not use this with printers other than StrPrinter
 PRECEDENCE_VALUES = {
     "Equivalent": PRECEDENCE["Xor"],
     "Xor": PRECEDENCE["Xor"],
@@ -115,8 +116,9 @@ def precedence_UnevaluatedExpr(item):
 
 
 def precedence(item):
-    """
-    Returns the precedence of a given object.
+    """Returns the precedence of a given object.
+
+    This is the precedence for StrPrinter.
     """
     if hasattr(item, "precedence"):
         return item.precedence
@@ -134,19 +136,22 @@ def precedence(item):
 
 
 def precedence_traditional(item):
-    """
-    Returns the precedence of a given object according to the traditional rules
-    of mathematics. This is the precedence for the LaTeX and pretty printer.
+    """Returns the precedence of a given object according to the
+    traditional rules of mathematics.
+
+    This is the precedence for the LaTeX and pretty printer.
     """
     # Integral, Sum, Product, Limit have the precedence of Mul in LaTeX,
     # the precedence of Atom for other printers:
-    from sympy import Integral, Sum, Product, Limit, Derivative
+    from sympy import Integral, Sum, Product, Limit, Derivative, Transpose, Adjoint
     from sympy.core.expr import UnevaluatedExpr
     from sympy.tensor.functions import TensorProduct
 
     if isinstance(item, (Integral, Sum, Product, Limit, Derivative, TensorProduct)):
         return PRECEDENCE["Mul"]
-    if (item.__class__.__name__ in ("Dot", "Cross", "Gradient", "Divergence",
+    elif isinstance(item, (Transpose, Adjoint)):
+        return PRECEDENCE["Pow"]
+    elif (item.__class__.__name__ in ("Dot", "Cross", "Gradient", "Divergence",
                                     "Curl", "Laplacian")):
         return PRECEDENCE["Mul"]-1
     elif isinstance(item, UnevaluatedExpr):

```

## Test Patch

```diff
diff --git a/sympy/printing/tests/test_latex.py b/sympy/printing/tests/test_latex.py
--- a/sympy/printing/tests/test_latex.py
+++ b/sympy/printing/tests/test_latex.py
@@ -1579,17 +1579,45 @@ def test_Adjoint():
     assert latex(Adjoint(Transpose(X))) == r'\left(X^{T}\right)^{\dagger}'
     assert latex(Transpose(Adjoint(X))) == r'\left(X^{\dagger}\right)^{T}'
     assert latex(Transpose(Adjoint(X) + Y)) == r'\left(X^{\dagger} + Y\right)^{T}'
+
+
+def test_Transpose():
+    from sympy.matrices import Transpose, MatPow, HadamardPower
+    X = MatrixSymbol('X', 2, 2)
+    Y = MatrixSymbol('Y', 2, 2)
     assert latex(Transpose(X)) == r'X^{T}'
     assert latex(Transpose(X + Y)) == r'\left(X + Y\right)^{T}'
 
+    assert latex(Transpose(HadamardPower(X, 2))) == \
+        r'\left(X^{\circ {2}}\right)^{T}'
+    assert latex(HadamardPower(Transpose(X), 2)) == \
+        r'\left(X^{T}\right)^{\circ {2}}'
+    assert latex(Transpose(MatPow(X, 2))) == \
+        r'\left(X^{2}\right)^{T}'
+    assert latex(MatPow(Transpose(X), 2)) == \
+        r'\left(X^{T}\right)^{2}'
+
 
 def test_Hadamard():
-    from sympy.matrices import MatrixSymbol, HadamardProduct
+    from sympy.matrices import MatrixSymbol, HadamardProduct, HadamardPower
+    from sympy.matrices.expressions import MatAdd, MatMul, MatPow
     X = MatrixSymbol('X', 2, 2)
     Y = MatrixSymbol('Y', 2, 2)
     assert latex(HadamardProduct(X, Y*Y)) == r'X \circ Y^{2}'
     assert latex(HadamardProduct(X, Y)*Y) == r'\left(X \circ Y\right) Y'
 
+    assert latex(HadamardPower(X, 2)) == r'X^{\circ {2}}'
+    assert latex(HadamardPower(X, -1)) == r'X^{\circ {-1}}'
+    assert latex(HadamardPower(MatAdd(X, Y), 2)) == \
+        r'\left(X + Y\right)^{\circ {2}}'
+    assert latex(HadamardPower(MatMul(X, Y), 2)) == \
+        r'\left(X Y\right)^{\circ {2}}'
+
+    assert latex(HadamardPower(MatPow(X, -1), -1)) == \
+        r'\left(X^{-1}\right)^{\circ {-1}}'
+    assert latex(MatPow(HadamardPower(X, -1), -1)) == \
+        r'\left(X^{\circ {-1}}\right)^{-1}'
+
 
 def test_ZeroMatrix():
     from sympy import ZeroMatrix

```


## Code snippets

### 1 - sympy/printing/latex.py:

Start line: 1560, End line: 1591

```python
class LatexPrinter(Printer):

    def _print_HadamardProduct(self, expr):
        from sympy import Add, MatAdd, MatMul

        def parens(x):
            if isinstance(x, (Add, MatAdd, MatMul)):
                return r"\left(%s\right)" % self._print(x)
            return self._print(x)
        return r' \circ '.join(map(parens, expr.args))

    def _print_KroneckerProduct(self, expr):
        from sympy import Add, MatAdd, MatMul

        def parens(x):
            if isinstance(x, (Add, MatAdd, MatMul)):
                return r"\left(%s\right)" % self._print(x)
            return self._print(x)
        return r' \otimes '.join(map(parens, expr.args))

    def _print_MatPow(self, expr):
        base, exp = expr.base, expr.exp
        from sympy.matrices import MatrixSymbol
        if not isinstance(base, MatrixSymbol):
            return r"\left(%s\right)^{%s}" % (self._print(base),
                                              self._print(exp))
        else:
            return "%s^{%s}" % (self._print(base), self._print(exp))

    def _print_ZeroMatrix(self, Z):
        return r"\mathbb{0}"

    def _print_Identity(self, I):
        return r"\mathbb{I}"
```
### 2 - sympy/printing/pretty/pretty.py:

Start line: 861, End line: 897

```python
class PrettyPrinter(Printer):

    def _print_DotProduct(self, expr):
        args = list(expr.args)

        for i, a in enumerate(args):
            args[i] = self._print(a)
        return prettyForm.__mul__(*args)

    def _print_MatPow(self, expr):
        pform = self._print(expr.base)
        from sympy.matrices import MatrixSymbol
        if not isinstance(expr.base, MatrixSymbol):
            pform = prettyForm(*pform.parens())
        pform = pform**(self._print(expr.exp))
        return pform

    def _print_HadamardProduct(self, expr):
        from sympy import MatAdd, MatMul
        if self._use_unicode:
            delim = pretty_atom('Ring')
        else:
            delim = '.*'
        return self._print_seq(expr.args, None, None, delim,
                parenthesize=lambda x: isinstance(x, (MatAdd, MatMul)))

    def _print_KroneckerProduct(self, expr):
        from sympy import MatAdd, MatMul
        if self._use_unicode:
            delim = u' \N{N-ARY CIRCLED TIMES OPERATOR} '
        else:
            delim = ' x '
        return self._print_seq(expr.args, None, None, delim,
                parenthesize=lambda x: isinstance(x, (MatAdd, MatMul)))

    def _print_FunctionMatrix(self, X):
        D = self._print(X.lamda.expr)
        D = prettyForm(*D.parens('[', ']'))
        return D
```
### 3 - sympy/printing/latex.py:

Start line: 947, End line: 1002

```python
class LatexPrinter(Printer):

    def _print_And(self, e):
        args = sorted(e.args, key=default_sort_key)
        return self._print_LogOp(args, r"\wedge")

    def _print_Or(self, e):
        args = sorted(e.args, key=default_sort_key)
        return self._print_LogOp(args, r"\vee")

    def _print_Xor(self, e):
        args = sorted(e.args, key=default_sort_key)
        return self._print_LogOp(args, r"\veebar")

    def _print_Implies(self, e, altchar=None):
        return self._print_LogOp(e.args, altchar or r"\Rightarrow")

    def _print_Equivalent(self, e, altchar=None):
        args = sorted(e.args, key=default_sort_key)
        return self._print_LogOp(args, altchar or r"\Leftrightarrow")

    def _print_conjugate(self, expr, exp=None):
        tex = r"\overline{%s}" % self._print(expr.args[0])

        if exp is not None:
            return r"%s^{%s}" % (tex, exp)
        else:
            return tex

    def _print_polar_lift(self, expr, exp=None):
        func = r"\operatorname{polar\_lift}"
        arg = r"{\left(%s \right)}" % self._print(expr.args[0])

        if exp is not None:
            return r"%s^{%s}%s" % (func, exp, arg)
        else:
            return r"%s%s" % (func, arg)

    def _print_ExpBase(self, expr, exp=None):
        # TODO should exp_polar be printed differently?
        #      what about exp_polar(0), exp_polar(1)?
        tex = r"e^{%s}" % self._print(expr.args[0])
        return self._do_exponent(tex, exp)

    def _print_elliptic_k(self, expr, exp=None):
        tex = r"\left(%s\right)" % self._print(expr.args[0])
        if exp is not None:
            return r"K^{%s}%s" % (exp, tex)
        else:
            return r"K%s" % tex

    def _print_elliptic_f(self, expr, exp=None):
        tex = r"\left(%s\middle| %s\right)" % \
            (self._print(expr.args[0]), self._print(expr.args[1]))
        if exp is not None:
            return r"F^{%s}%s" % (exp, tex)
        else:
            return r"F%s" % tex
```
### 4 - sympy/printing/latex.py:

Start line: 518, End line: 572

```python
class LatexPrinter(Printer):

    def _print_Pow(self, expr):
        # Treat x**Rational(1,n) as special case
        if expr.exp.is_Rational and abs(expr.exp.p) == 1 and expr.exp.q != 1 \
                and self._settings['root_notation']:
            base = self._print(expr.base)
            expq = expr.exp.q

            if expq == 2:
                tex = r"\sqrt{%s}" % base
            elif self._settings['itex']:
                tex = r"\root{%d}{%s}" % (expq, base)
            else:
                tex = r"\sqrt[%d]{%s}" % (expq, base)

            if expr.exp.is_negative:
                return r"\frac{1}{%s}" % tex
            else:
                return tex
        elif self._settings['fold_frac_powers'] \
            and expr.exp.is_Rational \
                and expr.exp.q != 1:
            base = self.parenthesize(expr.base, PRECEDENCE['Pow'])
            p, q = expr.exp.p, expr.exp.q
            # issue #12886: add parentheses for superscripts raised to powers
            if '^' in base and expr.base.is_Symbol:
                base = r"\left(%s\right)" % base
            if expr.base.is_Function:
                return self._print(expr.base, exp="%s/%s" % (p, q))
            return r"%s^{%s/%s}" % (base, p, q)
        elif expr.exp.is_Rational and expr.exp.is_negative and \
                expr.base.is_commutative:
            # special case for 1^(-x), issue 9216
            if expr.base == 1:
                return r"%s^{%s}" % (expr.base, expr.exp)
            # things like 1/x
            return self._print_Mul(expr)
        else:
            if expr.base.is_Function:
                return self._print(expr.base, exp=self._print(expr.exp))
            else:
                tex = r"%s^{%s}"
                exp = self._print(expr.exp)
                # issue #12886: add parentheses around superscripts raised
                # to powers
                base = self.parenthesize(expr.base, PRECEDENCE['Pow'])
                if '^' in base and expr.base.is_Symbol:
                    base = r"\left(%s\right)" % base
                elif (isinstance(expr.base, Derivative)
                        and base.startswith(r'\left(')
                        and re.match(r'\\left\(\\d?d?dot', base)
                        and base.endswith(r'\right)')):
                    # don't use parentheses around dotted derivative
                    base = base[6: -7]  # remove outermost added parens

                return tex % (base, exp)
```
### 5 - sympy/printing/pretty/pretty.py:

Start line: 849, End line: 859

```python
class PrettyPrinter(Printer):

    def _print_MatMul(self, expr):
        args = list(expr.args)
        from sympy import Add, MatAdd, HadamardProduct, KroneckerProduct
        for i, a in enumerate(args):
            if (isinstance(a, (Add, MatAdd, HadamardProduct, KroneckerProduct))
                    and len(expr.args) > 1):
                args[i] = prettyForm(*self._print(a).parens())
            else:
                args[i] = self._print(a)

        return prettyForm.__mul__(*args)
```
### 6 - sympy/printing/latex.py:

Start line: 839, End line: 919

```python
class LatexPrinter(Printer):

    def _print_FunctionClass(self, expr):
        for cls in self._special_function_classes:
            if issubclass(expr, cls) and expr.__name__ == cls.__name__:
                return self._special_function_classes[cls]
        return self._hprint_Function(str(expr))

    def _print_Lambda(self, expr):
        symbols, expr = expr.args

        if len(symbols) == 1:
            symbols = self._print(symbols[0])
        else:
            symbols = self._print(tuple(symbols))

        tex = r"\left( %s \mapsto %s \right)" % (symbols, self._print(expr))

        return tex

    def _hprint_variadic_function(self, expr, exp=None):
        args = sorted(expr.args, key=default_sort_key)
        texargs = [r"%s" % self._print(symbol) for symbol in args]
        tex = r"\%s\left(%s\right)" % (self._print((str(expr.func)).lower()),
                                       ", ".join(texargs))
        if exp is not None:
            return r"%s^{%s}" % (tex, exp)
        else:
            return tex

    _print_Min = _print_Max = _hprint_variadic_function

    def _print_floor(self, expr, exp=None):
        tex = r"\left\lfloor{%s}\right\rfloor" % self._print(expr.args[0])

        if exp is not None:
            return r"%s^{%s}" % (tex, exp)
        else:
            return tex

    def _print_ceiling(self, expr, exp=None):
        tex = r"\left\lceil{%s}\right\rceil" % self._print(expr.args[0])

        if exp is not None:
            return r"%s^{%s}" % (tex, exp)
        else:
            return tex

    def _print_log(self, expr, exp=None):
        if not self._settings["ln_notation"]:
            tex = r"\log{\left(%s \right)}" % self._print(expr.args[0])
        else:
            tex = r"\ln{\left(%s \right)}" % self._print(expr.args[0])

        if exp is not None:
            return r"%s^{%s}" % (tex, exp)
        else:
            return tex

    def _print_Abs(self, expr, exp=None):
        tex = r"\left|{%s}\right|" % self._print(expr.args[0])

        if exp is not None:
            return r"%s^{%s}" % (tex, exp)
        else:
            return tex
    _print_Determinant = _print_Abs

    def _print_re(self, expr, exp=None):
        if self._settings['gothic_re_im']:
            tex = r"\Re{%s}" % self.parenthesize(expr.args[0], PRECEDENCE['Atom'])
        else:
            tex = r"\operatorname{{re}}{{{}}}".format(self.parenthesize(expr.args[0], PRECEDENCE['Atom']))

        return self._do_exponent(tex, exp)

    def _print_im(self, expr, exp=None):
        if self._settings['gothic_re_im']:
            tex = r"\Im{%s}" % self.parenthesize(expr.args[0], PRECEDENCE['Atom'])
        else:
            tex = r"\operatorname{{im}}{{{}}}".format(self.parenthesize(expr.args[0], PRECEDENCE['Atom']))

        return self._do_exponent(tex, exp)
```
### 7 - sympy/printing/latex.py:

Start line: 1028, End line: 1112

```python
class LatexPrinter(Printer):

    def _print_beta(self, expr, exp=None):
        tex = r"\left(%s, %s\right)" % (self._print(expr.args[0]),
                                        self._print(expr.args[1]))

        if exp is not None:
            return r"\operatorname{B}^{%s}%s" % (exp, tex)
        else:
            return r"\operatorname{B}%s" % tex

    def _print_uppergamma(self, expr, exp=None):
        tex = r"\left(%s, %s\right)" % (self._print(expr.args[0]),
                                        self._print(expr.args[1]))

        if exp is not None:
            return r"\Gamma^{%s}%s" % (exp, tex)
        else:
            return r"\Gamma%s" % tex

    def _print_lowergamma(self, expr, exp=None):
        tex = r"\left(%s, %s\right)" % (self._print(expr.args[0]),
                                        self._print(expr.args[1]))

        if exp is not None:
            return r"\gamma^{%s}%s" % (exp, tex)
        else:
            return r"\gamma%s" % tex

    def _hprint_one_arg_func(self, expr, exp=None):
        tex = r"\left(%s\right)" % self._print(expr.args[0])

        if exp is not None:
            return r"%s^{%s}%s" % (self._print(expr.func), exp, tex)
        else:
            return r"%s%s" % (self._print(expr.func), tex)

    _print_gamma = _hprint_one_arg_func

    def _print_Chi(self, expr, exp=None):
        tex = r"\left(%s\right)" % self._print(expr.args[0])

        if exp is not None:
            return r"\operatorname{Chi}^{%s}%s" % (exp, tex)
        else:
            return r"\operatorname{Chi}%s" % tex

    def _print_expint(self, expr, exp=None):
        tex = r"\left(%s\right)" % self._print(expr.args[1])
        nu = self._print(expr.args[0])

        if exp is not None:
            return r"\operatorname{E}_{%s}^{%s}%s" % (nu, exp, tex)
        else:
            return r"\operatorname{E}_{%s}%s" % (nu, tex)

    def _print_fresnels(self, expr, exp=None):
        tex = r"\left(%s\right)" % self._print(expr.args[0])

        if exp is not None:
            return r"S^{%s}%s" % (exp, tex)
        else:
            return r"S%s" % tex

    def _print_fresnelc(self, expr, exp=None):
        tex = r"\left(%s\right)" % self._print(expr.args[0])

        if exp is not None:
            return r"C^{%s}%s" % (exp, tex)
        else:
            return r"C%s" % tex

    def _print_subfactorial(self, expr, exp=None):
        tex = r"!%s" % self.parenthesize(expr.args[0], PRECEDENCE["Func"])

        if exp is not None:
            return r"\left(%s\right)^{%s}" % (tex, exp)
        else:
            return tex

    def _print_factorial(self, expr, exp=None):
        tex = r"%s!" % self.parenthesize(expr.args[0], PRECEDENCE["Func"])

        if exp is not None:
            return r"%s^{%s}" % (tex, exp)
        else:
            return tex
```
### 8 - sympy/printing/latex.py:

Start line: 2222, End line: 2288

```python
class LatexPrinter(Printer):

    def _print_FreeModule(self, M):
        return '{{{}}}^{{{}}}'.format(self._print(M.ring), self._print(M.rank))

    def _print_FreeModuleElement(self, m):
        # Print as row vector for convenience, for now.
        return r"\left[ {} \right]".format(",".join(
            '{' + self._print(x) + '}' for x in m))

    def _print_SubModule(self, m):
        return r"\left\langle {} \right\rangle".format(",".join(
            '{' + self._print(x) + '}' for x in m.gens))

    def _print_ModuleImplementedIdeal(self, m):
        return r"\left\langle {} \right\rangle".format(",".join(
            '{' + self._print(x) + '}' for [x] in m._module.gens))

    def _print_Quaternion(self, expr):
        # TODO: This expression is potentially confusing,
        # shall we print it as `Quaternion( ... )`?
        s = [self.parenthesize(i, PRECEDENCE["Mul"], strict=True)
             for i in expr.args]
        a = [s[0]] + [i+" "+j for i, j in zip(s[1:], "ijk")]
        return " + ".join(a)

    def _print_QuotientRing(self, R):
        # TODO nicer fractions for few generators...
        return r"\frac{{{}}}{{{}}}".format(self._print(R.ring),
                 self._print(R.base_ideal))

    def _print_QuotientRingElement(self, x):
        return r"{{{}}} + {{{}}}".format(self._print(x.data),
                 self._print(x.ring.base_ideal))

    def _print_QuotientModuleElement(self, m):
        return r"{{{}}} + {{{}}}".format(self._print(m.data),
                 self._print(m.module.killed_module))

    def _print_QuotientModule(self, M):
        # TODO nicer fractions for few generators...
        return r"\frac{{{}}}{{{}}}".format(self._print(M.base),
                 self._print(M.killed_module))

    def _print_MatrixHomomorphism(self, h):
        return r"{{{}}} : {{{}}} \to {{{}}}".format(self._print(h._sympy_matrix()),
            self._print(h.domain), self._print(h.codomain))

    def _print_BaseScalarField(self, field):
        string = field._coord_sys._names[field._index]
        return r'\mathbf{{{}}}'.format(self._print(Symbol(string)))

    def _print_BaseVectorField(self, field):
        string = field._coord_sys._names[field._index]
        return r'\partial_{{{}}}'.format(self._print(Symbol(string)))

    def _print_Differential(self, diff):
        field = diff._form_field
        if hasattr(field, '_coord_sys'):
            string = field._coord_sys._names[field._index]
            return r'\operatorname{{d}}{}'.format(self._print(Symbol(string)))
        else:
            string = self._print(field)
            return r'\operatorname{{d}}\left({}\right)'.format(string)

    def _print_Tr(self, p):
        # TODO: Handle indices
        contents = self._print(p.args[0])
        return r'\operatorname{{tr}}\left({}\right)'.format(contents)
```
### 9 - sympy/printing/latex.py:

Start line: 1164, End line: 1229

```python
class LatexPrinter(Printer):

    def _hprint_vec(self, vec):
        if not vec:
            return ""
        s = ""
        for i in vec[:-1]:
            s += "%s, " % self._print(i)
        s += self._print(vec[-1])
        return s

    def _print_besselj(self, expr, exp=None):
        return self._hprint_BesselBase(expr, exp, 'J')

    def _print_besseli(self, expr, exp=None):
        return self._hprint_BesselBase(expr, exp, 'I')

    def _print_besselk(self, expr, exp=None):
        return self._hprint_BesselBase(expr, exp, 'K')

    def _print_bessely(self, expr, exp=None):
        return self._hprint_BesselBase(expr, exp, 'Y')

    def _print_yn(self, expr, exp=None):
        return self._hprint_BesselBase(expr, exp, 'y')

    def _print_jn(self, expr, exp=None):
        return self._hprint_BesselBase(expr, exp, 'j')

    def _print_hankel1(self, expr, exp=None):
        return self._hprint_BesselBase(expr, exp, 'H^{(1)}')

    def _print_hankel2(self, expr, exp=None):
        return self._hprint_BesselBase(expr, exp, 'H^{(2)}')

    def _print_hn1(self, expr, exp=None):
        return self._hprint_BesselBase(expr, exp, 'h^{(1)}')

    def _print_hn2(self, expr, exp=None):
        return self._hprint_BesselBase(expr, exp, 'h^{(2)}')

    def _hprint_airy(self, expr, exp=None, notation=""):
        tex = r"\left(%s\right)" % self._print(expr.args[0])

        if exp is not None:
            return r"%s^{%s}%s" % (notation, exp, tex)
        else:
            return r"%s%s" % (notation, tex)

    def _hprint_airy_prime(self, expr, exp=None, notation=""):
        tex = r"\left(%s\right)" % self._print(expr.args[0])

        if exp is not None:
            return r"{%s^\prime}^{%s}%s" % (notation, exp, tex)
        else:
            return r"%s^\prime%s" % (notation, tex)

    def _print_airyai(self, expr, exp=None):
        return self._hprint_airy(expr, exp, 'Ai')

    def _print_airybi(self, expr, exp=None):
        return self._hprint_airy(expr, exp, 'Bi')

    def _print_airyaiprime(self, expr, exp=None):
        return self._hprint_airy_prime(expr, exp, 'Ai')

    def _print_airybiprime(self, expr, exp=None):
        return self._hprint_airy_prime(expr, exp, 'Bi')
```
### 10 - sympy/printing/latex.py:

Start line: 417, End line: 467

```python
class LatexPrinter(Printer):

    def _print_Mul(self, expr):
        from sympy.core.power import Pow
        from sympy.physics.units import Quantity
        include_parens = False
        if _coeff_isneg(expr):
            expr = -expr
            tex = "- "
            if expr.is_Add:
                tex += "("
                include_parens = True
        else:
            tex = ""

        from sympy.simplify import fraction
        numer, denom = fraction(expr, exact=True)
        separator = self._settings['mul_symbol_latex']
        numbersep = self._settings['mul_symbol_latex_numbers']

        def convert(expr):
            if not expr.is_Mul:
                return str(self._print(expr))
            else:
                _tex = last_term_tex = ""

                if self.order not in ('old', 'none'):
                    args = expr.as_ordered_factors()
                else:
                    args = list(expr.args)

                # If quantities are present append them at the back
                args = sorted(args, key=lambda x: isinstance(x, Quantity) or
                              (isinstance(x, Pow) and
                               isinstance(x.base, Quantity)))

                for i, term in enumerate(args):
                    term_tex = self._print(term)

                    if self._needs_mul_brackets(term, first=(i == 0),
                                                last=(i == len(args) - 1)):
                        term_tex = r"\left(%s\right)" % term_tex

                    if _between_two_numbers_p[0].search(last_term_tex) and \
                            _between_two_numbers_p[1].match(term_tex):
                        # between two numbers
                        _tex += numbersep
                    elif _tex:
                        _tex += separator

                    _tex += term_tex
                    last_term_tex = term_tex
                return _tex
        # ... other code
```
### 11 - sympy/printing/latex.py:

Start line: 1, End line: 83

```python
"""
A Printer which converts an expression into its LaTeX equivalent.
"""

from __future__ import print_function, division

import itertools

from sympy.core import S, Add, Symbol, Mod
from sympy.core.alphabets import greeks
from sympy.core.containers import Tuple
from sympy.core.function import _coeff_isneg, AppliedUndef, Derivative
from sympy.core.operations import AssocOp
from sympy.core.sympify import SympifyError
from sympy.logic.boolalg import true

# sympy.printing imports
from sympy.printing.precedence import precedence_traditional
from sympy.printing.printer import Printer
from sympy.printing.conventions import split_super_sub, requires_partial
from sympy.printing.precedence import precedence, PRECEDENCE

import mpmath.libmp as mlib
from mpmath.libmp import prec_to_dps

from sympy.core.compatibility import default_sort_key, range
from sympy.utilities.iterables import has_variety

import re

# Hand-picked functions which can be used directly in both LaTeX and MathJax
# Complete list at
# https://docs.mathjax.org/en/latest/tex.html#supported-latex-commands
# This variable only contains those functions which sympy uses.
accepted_latex_functions = ['arcsin', 'arccos', 'arctan', 'sin', 'cos', 'tan',
                            'sinh', 'cosh', 'tanh', 'sqrt', 'ln', 'log', 'sec',
                            'csc', 'cot', 'coth', 're', 'im', 'frac', 'root',
                            'arg',
                            ]

tex_greek_dictionary = {
    'Alpha': 'A',
    'Beta': 'B',
    'Gamma': r'\Gamma',
    'Delta': r'\Delta',
    'Epsilon': 'E',
    'Zeta': 'Z',
    'Eta': 'H',
    'Theta': r'\Theta',
    'Iota': 'I',
    'Kappa': 'K',
    'Lambda': r'\Lambda',
    'Mu': 'M',
    'Nu': 'N',
    'Xi': r'\Xi',
    'omicron': 'o',
    'Omicron': 'O',
    'Pi': r'\Pi',
    'Rho': 'P',
    'Sigma': r'\Sigma',
    'Tau': 'T',
    'Upsilon': r'\Upsilon',
    'Phi': r'\Phi',
    'Chi': 'X',
    'Psi': r'\Psi',
    'Omega': r'\Omega',
    'lamda': r'\lambda',
    'Lamda': r'\Lambda',
    'khi': r'\chi',
    'Khi': r'X',
    'varepsilon': r'\varepsilon',
    'varkappa': r'\varkappa',
    'varphi': r'\varphi',
    'varpi': r'\varpi',
    'varrho': r'\varrho',
    'varsigma': r'\varsigma',
    'vartheta': r'\vartheta',
}

other_symbols = set(['aleph', 'beth', 'daleth', 'gimel', 'ell', 'eth', 'hbar',
                     'hslash', 'mho', 'wp', ])

# Variable name modifiers
```
### 12 - sympy/printing/latex.py:

Start line: 389, End line: 415

```python
class LatexPrinter(Printer):

    def _print_Cross(self, expr):
        vec1 = expr._expr1
        vec2 = expr._expr2
        return r"%s \times %s" % (self.parenthesize(vec1, PRECEDENCE['Mul']),
                                  self.parenthesize(vec2, PRECEDENCE['Mul']))

    def _print_Curl(self, expr):
        vec = expr._expr
        return r"\nabla\times %s" % self.parenthesize(vec, PRECEDENCE['Mul'])

    def _print_Divergence(self, expr):
        vec = expr._expr
        return r"\nabla\cdot %s" % self.parenthesize(vec, PRECEDENCE['Mul'])

    def _print_Dot(self, expr):
        vec1 = expr._expr1
        vec2 = expr._expr2
        return r"%s \cdot %s" % (self.parenthesize(vec1, PRECEDENCE['Mul']),
                                 self.parenthesize(vec2, PRECEDENCE['Mul']))

    def _print_Gradient(self, expr):
        func = expr._expr
        return r"\nabla %s" % self.parenthesize(func, PRECEDENCE['Mul'])

    def _print_Laplacian(self, expr):
        func = expr._expr
        return r"\triangle %s" % self.parenthesize(func, PRECEDENCE['Mul'])
```
### 13 - sympy/printing/latex.py:

Start line: 2109, End line: 2190

```python
class LatexPrinter(Printer):

    def _print_catalan(self, expr, exp=None):
        tex = r"C_{%s}" % self._print(expr.args[0])
        if exp is not None:
            tex = r"%s^{%s}" % (tex, self._print(exp))
        return tex

    def _print_UnifiedTransform(self, expr, s, inverse=False):
        return r"\mathcal{{{}}}{}_{{{}}}\left[{}\right]\left({}\right)".format(s, '^{-1}' if inverse else '', self._print(expr.args[1]), self._print(expr.args[0]), self._print(expr.args[2]))

    def _print_MellinTransform(self, expr):
        return self._print_UnifiedTransform(expr, 'M')

    def _print_InverseMellinTransform(self, expr):
        return self._print_UnifiedTransform(expr, 'M', True)

    def _print_LaplaceTransform(self, expr):
        return self._print_UnifiedTransform(expr, 'L')

    def _print_InverseLaplaceTransform(self, expr):
        return self._print_UnifiedTransform(expr, 'L', True)

    def _print_FourierTransform(self, expr):
        return self._print_UnifiedTransform(expr, 'F')

    def _print_InverseFourierTransform(self, expr):
        return self._print_UnifiedTransform(expr, 'F', True)

    def _print_SineTransform(self, expr):
        return self._print_UnifiedTransform(expr, 'SIN')

    def _print_InverseSineTransform(self, expr):
        return self._print_UnifiedTransform(expr, 'SIN', True)

    def _print_CosineTransform(self, expr):
        return self._print_UnifiedTransform(expr, 'COS')

    def _print_InverseCosineTransform(self, expr):
        return self._print_UnifiedTransform(expr, 'COS', True)

    def _print_DMP(self, p):
        try:
            if p.ring is not None:
                # TODO incorporate order
                return self._print(p.ring.to_sympy(p))
        except SympifyError:
            pass
        return self._print(repr(p))

    def _print_DMF(self, p):
        return self._print_DMP(p)

    def _print_Object(self, object):
        return self._print(Symbol(object.name))

    def _print_Morphism(self, morphism):
        domain = self._print(morphism.domain)
        codomain = self._print(morphism.codomain)
        return "%s\\rightarrow %s" % (domain, codomain)

    def _print_NamedMorphism(self, morphism):
        pretty_name = self._print(Symbol(morphism.name))
        pretty_morphism = self._print_Morphism(morphism)
        return "%s:%s" % (pretty_name, pretty_morphism)

    def _print_IdentityMorphism(self, morphism):
        from sympy.categories import NamedMorphism
        return self._print_NamedMorphism(NamedMorphism(
            morphism.domain, morphism.codomain, "id"))

    def _print_CompositeMorphism(self, morphism):
        # All components of the morphism have names and it is thus
        # possible to build the name of the composite.
        component_names_list = [self._print(Symbol(component.name)) for
                                component in morphism.components]
        component_names_list.reverse()
        component_names = "\\circ ".join(component_names_list) + ":"

        pretty_morphism = self._print_Morphism(morphism)
        return component_names + pretty_morphism

    def _print_Category(self, morphism):
        return r"\mathbf{{{}}}".format(self._print(Symbol(morphism.name)))
```
### 15 - sympy/printing/latex.py:

Start line: 596, End line: 613

```python
class LatexPrinter(Printer):

    def _print_Product(self, expr):
        if len(expr.limits) == 1:
            tex = r"\prod_{%s=%s}^{%s} " % \
                tuple([self._print(i) for i in expr.limits[0]])
        else:
            def _format_ineq(l):
                return r"%s \leq %s \leq %s" % \
                    tuple([self._print(s) for s in (l[1], l[0], l[2])])

            tex = r"\prod_{\substack{%s}} " % \
                str.join('\\\\', [_format_ineq(l) for l in expr.limits])

        if isinstance(expr.function, Add):
            tex += r"\left(%s\right)" % self._print(expr.function)
        else:
            tex += self._print(expr.function)

        return tex
```
### 16 - sympy/printing/latex.py:

Start line: 1881, End line: 1943

```python
class LatexPrinter(Printer):

    _print_SeqPer = _print_SeqFormula
    _print_SeqAdd = _print_SeqFormula
    _print_SeqMul = _print_SeqFormula

    def _print_Interval(self, i):
        if i.start == i.end:
            return r"\left\{%s\right\}" % self._print(i.start)

        else:
            if i.left_open:
                left = '('
            else:
                left = '['

            if i.right_open:
                right = ')'
            else:
                right = ']'

            return r"\left%s%s, %s\right%s" % \
                   (left, self._print(i.start), self._print(i.end), right)

    def _print_AccumulationBounds(self, i):
        return r"\left\langle %s, %s\right\rangle" % \
                (self._print(i.min), self._print(i.max))

    def _print_Union(self, u):
        return r" \cup ".join([self._print(i) for i in u.args])

    def _print_Complement(self, u):
        return r" \setminus ".join([self._print(i) for i in u.args])

    def _print_Intersection(self, u):
        return r" \cap ".join([self._print(i) for i in u.args])

    def _print_SymmetricDifference(self, u):
        return r" \triangle ".join([self._print(i) for i in u.args])

    def _print_EmptySet(self, e):
        return r"\emptyset"

    def _print_Naturals(self, n):
        return r"\mathbb{N}"

    def _print_Naturals0(self, n):
        return r"\mathbb{N}_0"

    def _print_Integers(self, i):
        return r"\mathbb{Z}"

    def _print_Reals(self, i):
        return r"\mathbb{R}"

    def _print_Complexes(self, i):
        return r"\mathbb{C}"

    def _print_ImageSet(self, s):
        sets = s.args[1:]
        varsets = [r"%s \in %s" % (self._print(var), self._print(setv))
                   for var, setv in zip(s.lamda.variables, sets)]
        return r"\left\{%s\; |\; %s\right\}" % (
            self._print(s.lamda.expr),
            ', '.join(varsets))
```
### 18 - sympy/printing/latex.py:

Start line: 1311, End line: 1371

```python
class LatexPrinter(Printer):

    def _print_legendre(self, expr, exp=None):
        n, x = map(self._print, expr.args)
        tex = r"P_{%s}\left(%s\right)" % (n, x)
        if exp is not None:
            tex = r"\left(" + tex + r"\right)^{%s}" % (self._print(exp))
        return tex

    def _print_assoc_legendre(self, expr, exp=None):
        n, a, x = map(self._print, expr.args)
        tex = r"P_{%s}^{\left(%s\right)}\left(%s\right)" % (n, a, x)
        if exp is not None:
            tex = r"\left(" + tex + r"\right)^{%s}" % (self._print(exp))
        return tex

    def _print_hermite(self, expr, exp=None):
        n, x = map(self._print, expr.args)
        tex = r"H_{%s}\left(%s\right)" % (n, x)
        if exp is not None:
            tex = r"\left(" + tex + r"\right)^{%s}" % (self._print(exp))
        return tex

    def _print_laguerre(self, expr, exp=None):
        n, x = map(self._print, expr.args)
        tex = r"L_{%s}\left(%s\right)" % (n, x)
        if exp is not None:
            tex = r"\left(" + tex + r"\right)^{%s}" % (self._print(exp))
        return tex

    def _print_assoc_laguerre(self, expr, exp=None):
        n, a, x = map(self._print, expr.args)
        tex = r"L_{%s}^{\left(%s\right)}\left(%s\right)" % (n, a, x)
        if exp is not None:
            tex = r"\left(" + tex + r"\right)^{%s}" % (self._print(exp))
        return tex

    def _print_Ynm(self, expr, exp=None):
        n, m, theta, phi = map(self._print, expr.args)
        tex = r"Y_{%s}^{%s}\left(%s,%s\right)" % (n, m, theta, phi)
        if exp is not None:
            tex = r"\left(" + tex + r"\right)^{%s}" % (self._print(exp))
        return tex

    def _print_Znm(self, expr, exp=None):
        n, m, theta, phi = map(self._print, expr.args)
        tex = r"Z_{%s}^{%s}\left(%s,%s\right)" % (n, m, theta, phi)
        if exp is not None:
            tex = r"\left(" + tex + r"\right)^{%s}" % (self._print(exp))
        return tex

    def _print_Rational(self, expr):
        if expr.q != 1:
            sign = ""
            p = expr.p
            if expr.p < 0:
                sign = "- "
                p = -p
            if self._settings['fold_short_frac']:
                return r"%s%d / %d" % (sign, p, expr.q)
            return r"%s\frac{%d}{%d}" % (sign, p, expr.q)
        else:
            return self._print(expr.p)
```
### 20 - sympy/printing/latex.py:

Start line: 1957, End line: 2004

```python
class LatexPrinter(Printer):

    def _print_ComplexRegion(self, s):
        vars_print = ', '.join([self._print(var) for var in s.variables])
        return r"\left\{%s\; |\; %s \in %s \right\}" % (
            self._print(s.expr),
            vars_print,
            self._print(s.sets))

    def _print_Contains(self, e):
        return r"%s \in %s" % tuple(self._print(a) for a in e.args)

    def _print_FourierSeries(self, s):
        return self._print_Add(s.truncate()) + self._print(r' + \ldots')

    def _print_FormalPowerSeries(self, s):
        return self._print_Add(s.infinite)

    def _print_FiniteField(self, expr):
        return r"\mathbb{F}_{%s}" % expr.mod

    def _print_IntegerRing(self, expr):
        return r"\mathbb{Z}"

    def _print_RationalField(self, expr):
        return r"\mathbb{Q}"

    def _print_RealField(self, expr):
        return r"\mathbb{R}"

    def _print_ComplexField(self, expr):
        return r"\mathbb{C}"

    def _print_PolynomialRing(self, expr):
        domain = self._print(expr.domain)
        symbols = ", ".join(map(self._print, expr.symbols))
        return r"%s\left[%s\right]" % (domain, symbols)

    def _print_FractionField(self, expr):
        domain = self._print(expr.domain)
        symbols = ", ".join(map(self._print, expr.symbols))
        return r"%s\left(%s\right)" % (domain, symbols)

    def _print_PolynomialRingBase(self, expr):
        domain = self._print(expr.domain)
        symbols = ", ".join(map(self._print, expr.symbols))
        inv = ""
        if not expr.is_Poly:
            inv = r"S_<^{-1}"
        return r"%s%s\left[%s\right]" % (inv, domain, symbols)
```
### 24 - sympy/printing/latex.py:

Start line: 1255, End line: 1309

```python
class LatexPrinter(Printer):

    def _print_dirichlet_eta(self, expr, exp=None):
        tex = r"\left(%s\right)" % self._print(expr.args[0])
        if exp is not None:
            return r"\eta^{%s}%s" % (self._print(exp), tex)
        return r"\eta%s" % tex

    def _print_zeta(self, expr, exp=None):
        if len(expr.args) == 2:
            tex = r"\left(%s, %s\right)" % tuple(map(self._print, expr.args))
        else:
            tex = r"\left(%s\right)" % self._print(expr.args[0])
        if exp is not None:
            return r"\zeta^{%s}%s" % (self._print(exp), tex)
        return r"\zeta%s" % tex

    def _print_lerchphi(self, expr, exp=None):
        tex = r"\left(%s, %s, %s\right)" % tuple(map(self._print, expr.args))
        if exp is None:
            return r"\Phi%s" % tex
        return r"\Phi^{%s}%s" % (self._print(exp), tex)

    def _print_polylog(self, expr, exp=None):
        s, z = map(self._print, expr.args)
        tex = r"\left(%s\right)" % z
        if exp is None:
            return r"\operatorname{Li}_{%s}%s" % (s, tex)
        return r"\operatorname{Li}_{%s}^{%s}%s" % (s, self._print(exp), tex)

    def _print_jacobi(self, expr, exp=None):
        n, a, b, x = map(self._print, expr.args)
        tex = r"P_{%s}^{\left(%s,%s\right)}\left(%s\right)" % (n, a, b, x)
        if exp is not None:
            tex = r"\left(" + tex + r"\right)^{%s}" % (self._print(exp))
        return tex

    def _print_gegenbauer(self, expr, exp=None):
        n, a, x = map(self._print, expr.args)
        tex = r"C_{%s}^{\left(%s\right)}\left(%s\right)" % (n, a, x)
        if exp is not None:
            tex = r"\left(" + tex + r"\right)^{%s}" % (self._print(exp))
        return tex

    def _print_chebyshevt(self, expr, exp=None):
        n, x = map(self._print, expr.args)
        tex = r"T_{%s}\left(%s\right)" % (n, x)
        if exp is not None:
            tex = r"\left(" + tex + r"\right)^{%s}" % (self._print(exp))
        return tex

    def _print_chebyshevu(self, expr, exp=None):
        n, x = map(self._print, expr.args)
        tex = r"U_{%s}\left(%s\right)" % (n, x)
        if exp is not None:
            tex = r"\left(" + tex + r"\right)^{%s}" % (self._print(exp))
        return tex
```
### 25 - sympy/printing/latex.py:

Start line: 1754, End line: 1764

```python
class LatexPrinter(Printer):

    def _print_SingularityFunction(self, expr):
        shift = self._print(expr.args[0] - expr.args[1])
        power = self._print(expr.args[2])
        tex = r"{\left\langle %s \right\rangle}^{%s}" % (shift, power)
        return tex

    def _print_Heaviside(self, expr, exp=None):
        tex = r"\theta\left(%s\right)" % self._print(expr.args[0])
        if exp:
            tex = r"\left(%s\right)^{%s}" % (tex, exp)
        return tex
```
### 26 - sympy/printing/latex.py:

Start line: 1777, End line: 1791

```python
class LatexPrinter(Printer):

    def _print_LeviCivita(self, expr, exp=None):
        indices = map(self._print, expr.args)
        if all(x.is_Atom for x in expr.args):
            tex = r'\varepsilon_{%s}' % " ".join(indices)
        else:
            tex = r'\varepsilon_{%s}' % ", ".join(indices)
        if exp:
            tex = r'\left(%s\right)^{%s}' % (tex, exp)
        return tex

    def _print_ProductSet(self, p):
        if len(p.sets) > 1 and not has_variety(p.sets):
            return self._print(p.sets[0]) + "^{%d}" % len(p.sets)
        else:
            return r" \times ".join(self._print(set) for set in p.sets)
```
### 27 - sympy/printing/latex.py:

Start line: 469, End line: 516

```python
class LatexPrinter(Printer):

    def _print_Mul(self, expr):
        # ... other code

        if denom is S.One and Pow(1, -1, evaluate=False) not in expr.args:
            # use the original expression here, since fraction() may have
            # altered it when producing numer and denom
            tex += convert(expr)

        else:
            snumer = convert(numer)
            sdenom = convert(denom)
            ldenom = len(sdenom.split())
            ratio = self._settings['long_frac_ratio']
            if self._settings['fold_short_frac'] and ldenom <= 2 and \
                    "^" not in sdenom:
                # handle short fractions
                if self._needs_mul_brackets(numer, last=False):
                    tex += r"\left(%s\right) / %s" % (snumer, sdenom)
                else:
                    tex += r"%s / %s" % (snumer, sdenom)
            elif ratio is not None and \
                    len(snumer.split()) > ratio*ldenom:
                # handle long fractions
                if self._needs_mul_brackets(numer, last=True):
                    tex += r"\frac{1}{%s}%s\left(%s\right)" \
                        % (sdenom, separator, snumer)
                elif numer.is_Mul:
                    # split a long numerator
                    a = S.One
                    b = S.One
                    for x in numer.args:
                        if self._needs_mul_brackets(x, last=False) or \
                                len(convert(a*x).split()) > ratio*ldenom or \
                                (b.is_commutative is x.is_commutative is False):
                            b *= x
                        else:
                            a *= x
                    if self._needs_mul_brackets(b, last=True):
                        tex += r"\frac{%s}{%s}%s\left(%s\right)" \
                            % (convert(a), sdenom, separator, convert(b))
                    else:
                        tex += r"\frac{%s}{%s}%s%s" \
                            % (convert(a), sdenom, separator, convert(b))
                else:
                    tex += r"\frac{1}{%s}%s%s" % (sdenom, separator, snumer)
            else:
                tex += r"\frac{%s}{%s}" % (snumer, sdenom)

        if include_parens:
            tex += ")"
        return tex
```
### 29 - sympy/printing/latex.py:

Start line: 1678, End line: 1742

```python
class LatexPrinter(Printer):

    def _print_Tensor(self, expr):
        name = expr.args[0].args[0]
        indices = expr.get_indices()
        return self._printer_tensor_indices(name, indices)

    def _print_TensorElement(self, expr):
        name = expr.expr.args[0].args[0]
        indices = expr.expr.get_indices()
        index_map = expr.index_map
        return self._printer_tensor_indices(name, indices, index_map)

    def _print_TensMul(self, expr):
        # prints expressions like "A(a)", "3*A(a)", "(1+x)*A(a)"
        sign, args = expr._get_args_for_traditional_printer()
        return sign + "".join(
            [self.parenthesize(arg, precedence(expr)) for arg in args]
        )

    def _print_TensAdd(self, expr):
        a = []
        args = expr.args
        for x in args:
            a.append(self.parenthesize(x, precedence(expr)))
        a.sort()
        s = ' + '.join(a)
        s = s.replace('+ -', '- ')
        return s

    def _print_TensorIndex(self, expr):
        return "{}%s{%s}" % (
            "^" if expr.is_up else "_",
            self._print(expr.args[0])
        )

    def _print_tuple(self, expr):
        return r"\left( %s\right)" % \
            r", \  ".join([self._print(i) for i in expr])

    def _print_TensorProduct(self, expr):
        elements = [self._print(a) for a in expr.args]
        return r' \otimes '.join(elements)

    def _print_WedgeProduct(self, expr):
        elements = [self._print(a) for a in expr.args]
        return r' \wedge '.join(elements)

    def _print_Tuple(self, expr):
        return self._print_tuple(expr)

    def _print_list(self, expr):
        return r"\left[ %s\right]" % \
            r", \  ".join([self._print(i) for i in expr])

    def _print_dict(self, d):
        keys = sorted(d.keys(), key=default_sort_key)
        items = []

        for key in keys:
            val = d[key]
            items.append("%s : %s" % (self._print(key), self._print(val)))

        return r"\left\{ %s\right\}" % r", \  ".join(items)

    def _print_Dict(self, expr):
        return self._print_dict(expr)
```
### 35 - sympy/printing/latex.py:

Start line: 122, End line: 206

```python
class LatexPrinter(Printer):
    printmethod = "_latex"

    _default_settings = {
        "fold_frac_powers": False,
        "fold_func_brackets": False,
        "fold_short_frac": None,
        "inv_trig_style": "abbreviated",
        "itex": False,
        "ln_notation": False,
        "long_frac_ratio": None,
        "mat_delim": "[",
        "mat_str": None,
        "mode": "plain",
        "mul_symbol": None,
        "order": None,
        "symbol_names": {},
        "root_notation": True,
        "mat_symbol_style": "plain",
        "imaginary_unit": "i",
        "gothic_re_im": False,
    }

    def __init__(self, settings=None):
        Printer.__init__(self, settings)

        if 'mode' in self._settings:
            valid_modes = ['inline', 'plain', 'equation',
                           'equation*']
            if self._settings['mode'] not in valid_modes:
                raise ValueError("'mode' must be one of 'inline', 'plain', "
                                 "'equation' or 'equation*'")

        if self._settings['fold_short_frac'] is None and \
                self._settings['mode'] == 'inline':
            self._settings['fold_short_frac'] = True

        mul_symbol_table = {
            None: r" ",
            "ldot": r" \,.\, ",
            "dot": r" \cdot ",
            "times": r" \times "
        }
        try:
            self._settings['mul_symbol_latex'] = \
                mul_symbol_table[self._settings['mul_symbol']]
        except KeyError:
            self._settings['mul_symbol_latex'] = \
                self._settings['mul_symbol']
        try:
            self._settings['mul_symbol_latex_numbers'] = \
                mul_symbol_table[self._settings['mul_symbol'] or 'dot']
        except KeyError:
            if (self._settings['mul_symbol'].strip() in
                    ['', ' ', '\\', '\\,', '\\:', '\\;', '\\quad']):
                self._settings['mul_symbol_latex_numbers'] = \
                    mul_symbol_table['dot']
            else:
                self._settings['mul_symbol_latex_numbers'] = \
                    self._settings['mul_symbol']

        self._delim_dict = {'(': ')', '[': ']'}

        imaginary_unit_table = {
            None: r"i",
            "i": r"i",
            "ri": r"\mathrm{i}",
            "ti": r"\text{i}",
            "j": r"j",
            "rj": r"\mathrm{j}",
            "tj": r"\text{j}",
        }
        try:
            self._settings['imaginary_unit_latex'] = \
                imaginary_unit_table[self._settings['imaginary_unit']]
        except KeyError:
            self._settings['imaginary_unit_latex'] = \
                self._settings['imaginary_unit']

    def parenthesize(self, item, level, strict=False):
        prec_val = precedence_traditional(item)
        if (prec_val < level) or ((not strict) and prec_val <= level):
            return r"\left({}\right)".format(self._print(item))
        else:
            return self._print(item)
```
### 42 - sympy/printing/latex.py:

Start line: 1389, End line: 1423

```python
class LatexPrinter(Printer):

    def _print_Symbol(self, expr, style='plain'):
        if expr in self._settings['symbol_names']:
            return self._settings['symbol_names'][expr]

        result = self._deal_with_super_sub(expr.name) if \
            '\\' not in expr.name else expr.name

        if style == 'bold':
            result = r"\mathbf{{{}}}".format(result)

        return result

    _print_RandomSymbol = _print_Symbol

    def _print_MatrixSymbol(self, expr):
        return self._print_Symbol(expr,
                                  style=self._settings['mat_symbol_style'])

    def _deal_with_super_sub(self, string):
        if '{' in string:
            return string

        name, supers, subs = split_super_sub(string)

        name = translate(name)
        supers = [translate(sup) for sup in supers]
        subs = [translate(sub) for sub in subs]

        # glue all items together:
        if supers:
            name += "^{%s}" % " ".join(supers)
        if subs:
            name += "_{%s}" % " ".join(subs)

        return name
```
### 49 - sympy/printing/latex.py:

Start line: 2006, End line: 2061

```python
class LatexPrinter(Printer):

    def _print_Poly(self, poly):
        cls = poly.__class__.__name__
        terms = []
        for monom, coeff in poly.terms():
            s_monom = ''
            for i, exp in enumerate(monom):
                if exp > 0:
                    if exp == 1:
                        s_monom += self._print(poly.gens[i])
                    else:
                        s_monom += self._print(pow(poly.gens[i], exp))

            if coeff.is_Add:
                if s_monom:
                    s_coeff = r"\left(%s\right)" % self._print(coeff)
                else:
                    s_coeff = self._print(coeff)
            else:
                if s_monom:
                    if coeff is S.One:
                        terms.extend(['+', s_monom])
                        continue

                    if coeff is S.NegativeOne:
                        terms.extend(['-', s_monom])
                        continue

                s_coeff = self._print(coeff)

            if not s_monom:
                s_term = s_coeff
            else:
                s_term = s_coeff + " " + s_monom

            if s_term.startswith('-'):
                terms.extend(['-', s_term[1:]])
            else:
                terms.extend(['+', s_term])

        if terms[0] in ['-', '+']:
            modifier = terms.pop(0)

            if modifier == '-':
                terms[0] = '-' + terms[0]

        expr = ' '.join(terms)
        gens = list(map(self._print, poly.gens))
        domain = "domain=%s" % self._print(poly.get_domain())

        args = ", ".join([expr] + gens + [domain])
        if cls in accepted_latex_functions:
            tex = r"\%s {\left(%s \right)}" % (cls, args)
        else:
            tex = r"\operatorname{%s}{\left( %s \right)}" % (cls, args)

        return tex
```
### 50 - sympy/printing/latex.py:

Start line: 753, End line: 820

```python
class LatexPrinter(Printer):

    def _print_Function(self, expr, exp=None):
        r'''
        Render functions to LaTeX, handling functions that LaTeX knows about
        e.g., sin, cos, ... by using the proper LaTeX command (\sin, \cos, ...).
        For single-letter function names, render them as regular LaTeX math
        symbols. For multi-letter function names that LaTeX does not know
        about, (e.g., Li, sech) use \operatorname{} so that the function name
        is rendered in Roman font and LaTeX handles spacing properly.

        expr is the expression involving the function
        exp is an exponent
        '''
        func = expr.func.__name__
        if hasattr(self, '_print_' + func) and \
                not isinstance(expr, AppliedUndef):
            return getattr(self, '_print_' + func)(expr, exp)
        else:
            args = [str(self._print(arg)) for arg in expr.args]
            # How inverse trig functions should be displayed, formats are:
            # abbreviated: asin, full: arcsin, power: sin^-1
            inv_trig_style = self._settings['inv_trig_style']
            # If we are dealing with a power-style inverse trig function
            inv_trig_power_case = False
            # If it is applicable to fold the argument brackets
            can_fold_brackets = self._settings['fold_func_brackets'] and \
                len(args) == 1 and \
                not self._needs_function_brackets(expr.args[0])

            inv_trig_table = ["asin", "acos", "atan", "acsc", "asec", "acot"]

            # If the function is an inverse trig function, handle the style
            if func in inv_trig_table:
                if inv_trig_style == "abbreviated":
                    pass
                elif inv_trig_style == "full":
                    func = "arc" + func[1:]
                elif inv_trig_style == "power":
                    func = func[1:]
                    inv_trig_power_case = True

                    # Can never fold brackets if we're raised to a power
                    if exp is not None:
                        can_fold_brackets = False

            if inv_trig_power_case:
                if func in accepted_latex_functions:
                    name = r"\%s^{-1}" % func
                else:
                    name = r"\operatorname{%s}^{-1}" % func
            elif exp is not None:
                name = r'%s^{%s}' % (self._hprint_Function(func), exp)
            else:
                name = self._hprint_Function(func)

            if can_fold_brackets:
                if func in accepted_latex_functions:
                    # Wrap argument safely to avoid parse-time conflicts
                    # with the function name itself
                    name += r" {%s}"
                else:
                    name += r"%s"
            else:
                name += r"{\left(%s \right)}"

            if inv_trig_power_case and exp is not None:
                name += r"^{%s}" % exp

            return name % ",".join(args)
```
### 51 - sympy/printing/latex.py:

Start line: 1484, End line: 1504

```python
class LatexPrinter(Printer):
    _print_ImmutableMatrix = _print_ImmutableDenseMatrix \
                           = _print_Matrix \
                           = _print_MatrixBase

    def _print_MatrixElement(self, expr):
        return self.parenthesize(expr.parent, PRECEDENCE["Atom"], strict=True)\
            + '_{%s, %s}' % (self._print(expr.i), self._print(expr.j))

    def _print_MatrixSlice(self, expr):
        def latexslice(x):
            x = list(x)
            if x[2] == 1:
                del x[2]
            if x[1] == x[0] + 1:
                del x[1]
            if x[0] == 0:
                x[0] = ''
            return ':'.join(map(self._print, x))
        return (self._print(expr.parent) + r'\left[' +
                latexslice(expr.rowslice) + ', ' +
                latexslice(expr.colslice) + r'\right]')
```
### 52 - sympy/printing/latex.py:

Start line: 2088, End line: 2098

```python
class LatexPrinter(Printer):

    def _print_PolyElement(self, poly):
        mul_symbol = self._settings['mul_symbol_latex']
        return poly.str(self, PRECEDENCE, "{%s}^{%d}", mul_symbol)

    def _print_FracElement(self, frac):
        if frac.denom == 1:
            return self._print(frac.numer)
        else:
            numer = self._print(frac.numer)
            denom = self._print(frac.denom)
            return r"\frac{%s}{%s}" % (numer, denom)
```
### 59 - sympy/printing/latex.py:

Start line: 1529, End line: 1548

```python
class LatexPrinter(Printer):

    def _print_MatMul(self, expr):
        from sympy import MatMul, Mul

        parens = lambda x: self.parenthesize(x, precedence_traditional(expr),
                                             False)

        args = expr.args
        if isinstance(args[0], Mul):
            args = args[0].as_ordered_factors() + list(args[1:])
        else:
            args = list(args)

        if isinstance(expr, MatMul) and _coeff_isneg(expr):
            if args[0] == -1:
                args = args[1:]
            else:
                args[0] = -args[0]
            return '- ' + ' '.join(map(parens, args))
        else:
            return ' '.join(map(parens, args))
```
### 60 - sympy/printing/latex.py:

Start line: 1833, End line: 1857

```python
class LatexPrinter(Printer):

    def _print_bernoulli(self, expr, exp=None):
        tex = r"B_{%s}" % self._print(expr.args[0])
        if exp is not None:
            tex = r"%s^{%s}" % (tex, self._print(exp))
        return tex

    _print_bell = _print_bernoulli

    def _print_fibonacci(self, expr, exp=None):
        tex = r"F_{%s}" % self._print(expr.args[0])
        if exp is not None:
            tex = r"%s^{%s}" % (tex, self._print(exp))
        return tex

    def _print_lucas(self, expr, exp=None):
        tex = r"L_{%s}" % self._print(expr.args[0])
        if exp is not None:
            tex = r"%s^{%s}" % (tex, self._print(exp))
        return tex

    def _print_tribonacci(self, expr, exp=None):
        tex = r"T_{%s}" % self._print(expr.args[0])
        if exp is not None:
            tex = r"%s^{%s}" % (tex, self._print(exp))
        return tex
```
### 62 - sympy/printing/latex.py:

Start line: 366, End line: 387

```python
class LatexPrinter(Printer):

    def _print_Float(self, expr):
        # Based off of that in StrPrinter
        dps = prec_to_dps(expr._prec)
        str_real = mlib.to_str(expr._mpf_, dps, strip_zeros=True)

        # Must always have a mul symbol (as 2.5 10^{20} just looks odd)
        # thus we use the number separator
        separator = self._settings['mul_symbol_latex_numbers']

        if 'e' in str_real:
            (mant, exp) = str_real.split('e')

            if exp[0] == '+':
                exp = exp[1:]

            return r"%s%s10^{%s}" % (mant, separator, exp)
        elif str_real == "+inf":
            return r"\infty"
        elif str_real == "-inf":
            return r"- \infty"
        else:
            return str_real
```
### 64 - sympy/printing/latex.py:

Start line: 326, End line: 346

```python
class LatexPrinter(Printer):

    def _print_Add(self, expr, order=None):
        if self.order == 'none':
            terms = list(expr.args)
        else:
            terms = self._as_ordered_terms(expr, order=order)

        tex = ""
        for i, term in enumerate(terms):
            if i == 0:
                pass
            elif _coeff_isneg(term):
                tex += " - "
                term = -term
            else:
                tex += " + "
            term_tex = self._print(term)
            if self._needs_add_brackets(term):
                term_tex = r"\left(%s\right)" % term_tex
            tex += term_tex

        return tex
```
### 65 - sympy/printing/latex.py:

Start line: 645, End line: 676

```python
class LatexPrinter(Printer):

    def _print_Indexed(self, expr):
        tex_base = self._print(expr.base)
        tex = '{'+tex_base+'}'+'_{%s}' % ','.join(
            map(self._print, expr.indices))
        return tex

    def _print_IndexedBase(self, expr):
        return self._print(expr.label)

    def _print_Derivative(self, expr):
        if requires_partial(expr):
            diff_symbol = r'\partial'
        else:
            diff_symbol = r'd'

        tex = ""
        dim = 0
        for x, num in reversed(expr.variable_count):
            dim += num
            if num == 1:
                tex += r"%s %s" % (diff_symbol, self._print(x))
            else:
                tex += r"%s %s^{%s}" % (diff_symbol, self._print(x), num)

        if dim == 1:
            tex = r"\frac{%s}{%s}" % (diff_symbol, tex)
        else:
            tex = r"\frac{%s^{%s}}{%s}" % (diff_symbol, dim, tex)

        return r"%s %s" % (tex, self.parenthesize(expr.expr,
                                                  PRECEDENCE["Mul"],
                                                  strict=True))
```
### 66 - sympy/printing/latex.py:

Start line: 1114, End line: 1145

```python
class LatexPrinter(Printer):

    def _print_factorial2(self, expr, exp=None):
        tex = r"%s!!" % self.parenthesize(expr.args[0], PRECEDENCE["Func"])

        if exp is not None:
            return r"%s^{%s}" % (tex, exp)
        else:
            return tex

    def _print_binomial(self, expr, exp=None):
        tex = r"{\binom{%s}{%s}}" % (self._print(expr.args[0]),
                                     self._print(expr.args[1]))

        if exp is not None:
            return r"%s^{%s}" % (tex, exp)
        else:
            return tex

    def _print_RisingFactorial(self, expr, exp=None):
        n, k = expr.args
        base = r"%s" % self.parenthesize(n, PRECEDENCE['Func'])

        tex = r"{%s}^{\left(%s\right)}" % (base, self._print(k))

        return self._do_exponent(tex, exp)

    def _print_FallingFactorial(self, expr, exp=None):
        n, k = expr.args
        sub = r"%s" % self.parenthesize(k, PRECEDENCE['Func'])

        tex = r"{\left(%s\right)}_{%s}" % (self._print(n), sub)

        return self._do_exponent(tex, exp)
```
### 67 - sympy/printing/latex.py:

Start line: 1593, End line: 1643

```python
class LatexPrinter(Printer):

    def _print_NDimArray(self, expr):

        if expr.rank() == 0:
            return self._print(expr[()])

        mat_str = self._settings['mat_str']
        if mat_str is None:
            if self._settings['mode'] == 'inline':
                mat_str = 'smallmatrix'
            else:
                if (expr.rank() == 0) or (expr.shape[-1] <= 10):
                    mat_str = 'matrix'
                else:
                    mat_str = 'array'
        block_str = r'\begin{%MATSTR%}%s\end{%MATSTR%}'
        block_str = block_str.replace('%MATSTR%', mat_str)
        if self._settings['mat_delim']:
            left_delim = self._settings['mat_delim']
            right_delim = self._delim_dict[left_delim]
            block_str = r'\left' + left_delim + block_str + \
                        r'\right' + right_delim

        if expr.rank() == 0:
            return block_str % ""

        level_str = [[]] + [[] for i in range(expr.rank())]
        shape_ranges = [list(range(i)) for i in expr.shape]
        for outer_i in itertools.product(*shape_ranges):
            level_str[-1].append(self._print(expr[outer_i]))
            even = True
            for back_outer_i in range(expr.rank()-1, -1, -1):
                if len(level_str[back_outer_i+1]) < expr.shape[back_outer_i]:
                    break
                if even:
                    level_str[back_outer_i].append(
                        r" & ".join(level_str[back_outer_i+1]))
                else:
                    level_str[back_outer_i].append(
                        block_str % (r"\\".join(level_str[back_outer_i+1])))
                    if len(level_str[back_outer_i+1]) == 1:
                        level_str[back_outer_i][-1] = r"\left[" + \
                            level_str[back_outer_i][-1] + r"\right]"
                even = not even
                level_str[back_outer_i+1] = []

        out_str = level_str[0][0]

        if expr.rank() % 2 == 1:
            out_str = block_str % out_str

        return out_str
```
### 68 - sympy/printing/latex.py:

Start line: 1793, End line: 1813

```python
class LatexPrinter(Printer):

    def _print_RandomDomain(self, d):
        if hasattr(d, 'as_boolean'):
            return '\\text{Domain: }' + self._print(d.as_boolean())
        elif hasattr(d, 'set'):
            return ('\\text{Domain: }' + self._print(d.symbols) + '\\text{ in }' +
                    self._print(d.set))
        elif hasattr(d, 'symbols'):
            return '\\text{Domain on }' + self._print(d.symbols)
        else:
            return self._print(None)

    def _print_FiniteSet(self, s):
        items = sorted(s.args, key=default_sort_key)
        return self._print_set(items)

    def _print_set(self, s):
        items = sorted(s, key=default_sort_key)
        items = ", ".join(map(self._print, items))
        return r"\left\{%s\right\}" % items

    _print_frozenset = _print_set
```
### 69 - sympy/printing/latex.py:

Start line: 2371, End line: 2542

```python
def latex(expr, fold_frac_powers=False, fold_func_brackets=False,
          fold_short_frac=None, inv_trig_style="abbreviated",
          itex=False, ln_notation=False, long_frac_ratio=None,
          mat_delim="[", mat_str=None, mode="plain", mul_symbol=None,
          order=None, symbol_names=None, root_notation=True,
          mat_symbol_style="plain", imaginary_unit="i", gothic_re_im=False):
    r"""Convert the given expression to LaTeX string representation.

    Parameters
    ==========
    fold_frac_powers : boolean, optional
        Emit ``^{p/q}`` instead of ``^{\frac{p}{q}}`` for fractional powers.
    fold_func_brackets : boolean, optional
        Fold function brackets where applicable.
    fold_short_frac : boolean, optional
        Emit ``p / q`` instead of ``\frac{p}{q}`` when the denominator is
        simple enough (at most two terms and no powers). The default value is
        ``True`` for inline mode, ``False`` otherwise.
    inv_trig_style : string, optional
        How inverse trig functions should be displayed. Can be one of
        ``abbreviated``, ``full``, or ``power``. Defaults to ``abbreviated``.
    itex : boolean, optional
        Specifies if itex-specific syntax is used, including emitting
        ``$$...$$``.
    ln_notation : boolean, optional
        If set to ``True``, ``\ln`` is used instead of default ``\log``.
    long_frac_ratio : float or None, optional
        The allowed ratio of the width of the numerator to the width of the
        denominator before the printer breaks off long fractions. If ``None``
        (the default value), long fractions are not broken up.
    mat_delim : string, optional
        The delimiter to wrap around matrices. Can be one of ``[``, ``(``, or
        the empty string. Defaults to ``[``.
    mat_str : string, optional
        Which matrix environment string to emit. ``smallmatrix``, ``matrix``,
        ``array``, etc. Defaults to ``smallmatrix`` for inline mode, ``matrix``
        for matrices of no more than 10 columns, and ``array`` otherwise.
    mode: string, optional
        Specifies how the generated code will be delimited. ``mode`` can be one
        of ``plain``, ``inline``, ``equation`` or ``equation*``.  If ``mode``
        is set to ``plain``, then the resulting code will not be delimited at
        all (this is the default). If ``mode`` is set to ``inline`` then inline
        LaTeX ``$...$`` will be used. If ``mode`` is set to ``equation`` or
        ``equation*``, the resulting code will be enclosed in the ``equation``
        or ``equation*`` environment (remember to import ``amsmath`` for
        ``equation*``), unless the ``itex`` option is set. In the latter case,
        the ``$$...$$`` syntax is used.
    mul_symbol : string or None, optional
        The symbol to use for multiplication. Can be one of ``None``, ``ldot``,
        ``dot``, or ``times``.
    order: string, optional
        Any of the supported monomial orderings (currently ``lex``, ``grlex``,
        or ``grevlex``), ``old``, and ``none``. This parameter does nothing for
        Mul objects. Setting order to ``old`` uses the compatibility ordering
        for Add defined in Printer. For very large expressions, set the
        ``order`` keyword to ``none`` if speed is a concern.
    symbol_names : dictionary of strings mapped to symbols, optional
        Dictionary of symbols and the custom strings they should be emitted as.
    root_notation : boolean, optional
        If set to ``False``, exponents of the form 1/n are printed in fractonal
        form. Default is ``True``, to print exponent in root form.
    mat_symbol_style : string, optional
        Can be either ``plain`` (default) or ``bold``. If set to ``bold``,
        a MatrixSymbol A will be printed as ``\mathbf{A}``, otherwise as ``A``.
    imaginary_unit : string, optional
        String to use for the imaginary unit. Defined options are "i" (default)
        and "j". Adding "r" or "t" in front gives ``\mathrm`` or ``\text``, so
        "ri" leads to ``\mathrm{i}`` which gives `\mathrm{i}`.
    gothic_re_im : boolean, optional
        If set to ``True``, `\Re` and `\Im` is used for ``re`` and ``im``, respectively.
        The default is ``False`` leading to `\operatorname{re}` and `\operatorname{im}`.

    Notes
    =====

    Not using a print statement for printing, results in double backslashes for
    latex commands since that's the way Python escapes backslashes in strings.

    >>> from sympy import latex, Rational
    >>> from sympy.abc import tau
    >>> latex((2*tau)**Rational(7,2))
    '8 \\sqrt{2} \\tau^{\\frac{7}{2}}'
    >>> print(latex((2*tau)**Rational(7,2)))
    8 \sqrt{2} \tau^{\frac{7}{2}}

    Examples
    ========

    >>> from sympy import latex, pi, sin, asin, Integral, Matrix, Rational, log
    >>> from sympy.abc import x, y, mu, r, tau

    Basic usage:

    >>> print(latex((2*tau)**Rational(7,2)))
    8 \sqrt{2} \tau^{\frac{7}{2}}

    ``mode`` and ``itex`` options:

    >>> print(latex((2*mu)**Rational(7,2), mode='plain'))
    8 \sqrt{2} \mu^{\frac{7}{2}}
    >>> print(latex((2*tau)**Rational(7,2), mode='inline'))
    $8 \sqrt{2} \tau^{7 / 2}$
    >>> print(latex((2*mu)**Rational(7,2), mode='equation*'))
    \begin{equation*}8 \sqrt{2} \mu^{\frac{7}{2}}\end{equation*}
    >>> print(latex((2*mu)**Rational(7,2), mode='equation'))
    \begin{equation}8 \sqrt{2} \mu^{\frac{7}{2}}\end{equation}
    >>> print(latex((2*mu)**Rational(7,2), mode='equation', itex=True))
    $$8 \sqrt{2} \mu^{\frac{7}{2}}$$
    >>> print(latex((2*mu)**Rational(7,2), mode='plain'))
    8 \sqrt{2} \mu^{\frac{7}{2}}
    >>> print(latex((2*tau)**Rational(7,2), mode='inline'))
    $8 \sqrt{2} \tau^{7 / 2}$
    >>> print(latex((2*mu)**Rational(7,2), mode='equation*'))
    \begin{equation*}8 \sqrt{2} \mu^{\frac{7}{2}}\end{equation*}
    >>> print(latex((2*mu)**Rational(7,2), mode='equation'))
    \begin{equation}8 \sqrt{2} \mu^{\frac{7}{2}}\end{equation}
    >>> print(latex((2*mu)**Rational(7,2), mode='equation', itex=True))
    $$8 \sqrt{2} \mu^{\frac{7}{2}}$$

    Fraction options:

    >>> print(latex((2*tau)**Rational(7,2), fold_frac_powers=True))
    8 \sqrt{2} \tau^{7/2}
    >>> print(latex((2*tau)**sin(Rational(7,2))))
    \left(2 \tau\right)^{\sin{\left(\frac{7}{2} \right)}}
    >>> print(latex((2*tau)**sin(Rational(7,2)), fold_func_brackets=True))
    \left(2 \tau\right)^{\sin {\frac{7}{2}}}
    >>> print(latex(3*x**2/y))
    \frac{3 x^{2}}{y}
    >>> print(latex(3*x**2/y, fold_short_frac=True))
    3 x^{2} / y
    >>> print(latex(Integral(r, r)/2/pi, long_frac_ratio=2))
    \frac{\int r\, dr}{2 \pi}
    >>> print(latex(Integral(r, r)/2/pi, long_frac_ratio=0))
    \frac{1}{2 \pi} \int r\, dr

    Multiplication options:

    >>> print(latex((2*tau)**sin(Rational(7,2)), mul_symbol="times"))
    \left(2 \times \tau\right)^{\sin{\left(\frac{7}{2} \right)}}

    Trig options:

    >>> print(latex(asin(Rational(7,2))))
    \operatorname{asin}{\left(\frac{7}{2} \right)}
    >>> print(latex(asin(Rational(7,2)), inv_trig_style="full"))
    \arcsin{\left(\frac{7}{2} \right)}
    >>> print(latex(asin(Rational(7,2)), inv_trig_style="power"))
    \sin^{-1}{\left(\frac{7}{2} \right)}

    Matrix options:

    >>> print(latex(Matrix(2, 1, [x, y])))
    \left[\begin{matrix}x\\y\end{matrix}\right]
    >>> print(latex(Matrix(2, 1, [x, y]), mat_str = "array"))
    \left[\begin{array}{c}x\\y\end{array}\right]
    >>> print(latex(Matrix(2, 1, [x, y]), mat_delim="("))
    \left(\begin{matrix}x\\y\end{matrix}\right)

    Custom printing of symbols:

    >>> print(latex(x**2, symbol_names={x: 'x_i'}))
    x_i^{2}

    Logarithms:

    >>> print(latex(log(10)))
    \log{\left(10 \right)}
    >>> print(latex(log(10), ln_notation=True))
    \ln{\left(10 \right)}

    ``latex()`` also supports the builtin container types list, tuple, and
    dictionary.

    >>> print(latex([2/x, y], mode='inline'))
    $\left[ 2 / x, \  y\right]$

    """
    # ... other code
```
### 76 - sympy/printing/latex.py:

Start line: 1550, End line: 1558

```python
class LatexPrinter(Printer):

    def _print_Mod(self, expr, exp=None):
        if exp is not None:
            return r'\left(%s\bmod{%s}\right)^{%s}' % \
                (self.parenthesize(expr.args[0], PRECEDENCE['Mul'],
                                   strict=True), self._print(expr.args[1]),
                 self._print(exp))
        return r'%s\bmod{%s}' % (self.parenthesize(expr.args[0],
                                 PRECEDENCE['Mul'], strict=True),
                                 self._print(expr.args[1]))
```
### 78 - sympy/printing/latex.py:

Start line: 921, End line: 930

```python
class LatexPrinter(Printer):

    def _print_Not(self, e):
        from sympy import Equivalent, Implies
        if isinstance(e.args[0], Equivalent):
            return self._print_Equivalent(e.args[0], r"\not\Leftrightarrow")
        if isinstance(e.args[0], Implies):
            return self._print_Implies(e.args[0], r"\not\Rightarrow")
        if (e.args[0].is_Boolean):
            return r"\neg (%s)" % self._print(e.args[0])
        else:
            return r"\neg %s" % self._print(e.args[0])
```
### 79 - sympy/printing/latex.py:

Start line: 688, End line: 720

```python
class LatexPrinter(Printer):

    def _print_Integral(self, expr):
        tex, symbols = "", []

        # Only up to \iiiint exists
        if len(expr.limits) <= 4 and all(len(lim) == 1 for lim in expr.limits):
            # Use len(expr.limits)-1 so that syntax highlighters don't think
            # \" is an escaped quote
            tex = r"\i" + "i"*(len(expr.limits) - 1) + "nt"
            symbols = [r"\, d%s" % self._print(symbol[0])
                       for symbol in expr.limits]

        else:
            for lim in reversed(expr.limits):
                symbol = lim[0]
                tex += r"\int"

                if len(lim) > 1:
                    if self._settings['mode'] != 'inline' \
                            and not self._settings['itex']:
                        tex += r"\limits"

                    if len(lim) == 3:
                        tex += "_{%s}^{%s}" % (self._print(lim[1]),
                                               self._print(lim[2]))
                    if len(lim) == 2:
                        tex += "^{%s}" % (self._print(lim[1]))

                symbols.insert(0, r"\, d%s" % self._print(symbol))

        return r"%s %s%s" % (tex, self.parenthesize(expr.function,
                                                    PRECEDENCE["Mul"],
                                                    strict=True),
                             "".join(symbols))
```
### 82 - sympy/printing/latex.py:

Start line: 822, End line: 837

```python
class LatexPrinter(Printer):

    def _print_UndefinedFunction(self, expr):
        return self._hprint_Function(str(expr))

    @property
    def _special_function_classes(self):
        from sympy.functions.special.tensor_functions import KroneckerDelta
        from sympy.functions.special.gamma_functions import gamma, lowergamma
        from sympy.functions.special.beta_functions import beta
        from sympy.functions.special.delta_functions import DiracDelta
        from sympy.functions.special.error_functions import Chi
        return {KroneckerDelta: r'\delta',
                gamma:  r'\Gamma',
                lowergamma: r'\gamma',
                beta: r'\operatorname{B}',
                DiracDelta: r'\delta',
                Chi: r'\operatorname{Chi}'}
```
### 83 - sympy/printing/latex.py:

Start line: 1231, End line: 1240

```python
class LatexPrinter(Printer):

    def _print_hyper(self, expr, exp=None):
        tex = r"{{}_{%s}F_{%s}\left(\begin{matrix} %s \\ %s \end{matrix}" \
              r"\middle| {%s} \right)}" % \
            (self._print(len(expr.ap)), self._print(len(expr.bq)),
              self._hprint_vec(expr.ap), self._hprint_vec(expr.bq),
              self._print(expr.argument))

        if exp is not None:
            tex = r"{%s}^{%s}" % (tex, self._print(exp))
        return tex
```
### 84 - sympy/printing/latex.py:

Start line: 1242, End line: 1253

```python
class LatexPrinter(Printer):

    def _print_meijerg(self, expr, exp=None):
        tex = r"{G_{%s, %s}^{%s, %s}\left(\begin{matrix} %s & %s \\" \
              r"%s & %s \end{matrix} \middle| {%s} \right)}" % \
            (self._print(len(expr.ap)), self._print(len(expr.bq)),
              self._print(len(expr.bm)), self._print(len(expr.an)),
              self._hprint_vec(expr.an), self._hprint_vec(expr.aother),
              self._hprint_vec(expr.bm), self._hprint_vec(expr.bother),
              self._print(expr.argument))

        if exp is not None:
            tex = r"{%s}^{%s}" % (tex, self._print(exp))
        return tex
```
### 89 - sympy/printing/latex.py:

Start line: 1458, End line: 1483

```python
class LatexPrinter(Printer):

    def _print_MatrixBase(self, expr):
        lines = []

        for line in range(expr.rows):  # horrible, should be 'rows'
            lines.append(" & ".join([self._print(i) for i in expr[line, :]]))

        mat_str = self._settings['mat_str']
        if mat_str is None:
            if self._settings['mode'] == 'inline':
                mat_str = 'smallmatrix'
            else:
                if (expr.cols <= 10) is True:
                    mat_str = 'matrix'
                else:
                    mat_str = 'array'

        out_str = r'\begin{%MATSTR%}%s\end{%MATSTR%}'
        out_str = out_str.replace('%MATSTR%', mat_str)
        if mat_str == 'array':
            out_str = out_str.replace('%s', '{' + 'c'*expr.cols + '}%s')
        if self._settings['mat_delim']:
            left_delim = self._settings['mat_delim']
            right_delim = self._delim_dict[left_delim]
            out_str = r'\left' + left_delim + out_str + \
                      r'\right' + right_delim
        return out_str % r"\\".join(lines)
```
### 91 - sympy/printing/latex.py:

Start line: 2100, End line: 2107

```python
class LatexPrinter(Printer):

    def _print_euler(self, expr, exp=None):
        m, x = (expr.args[0], None) if len(expr.args) == 1 else expr.args
        tex = r"E_{%s}" % self._print(m)
        if exp is not None:
            tex = r"%s^{%s}" % (tex, self._print(exp))
        if x is not None:
            tex = r"%s\left(%s\right)" % (tex, self._print(x))
        return tex
```
### 95 - sympy/printing/latex.py:

Start line: 1373, End line: 1387

```python
class LatexPrinter(Printer):

    def _print_Order(self, expr):
        s = self._print(expr.expr)
        if expr.point and any(p != S.Zero for p in expr.point) or \
           len(expr.variables) > 1:
            s += '; '
            if len(expr.variables) > 1:
                s += self._print(expr.variables)
            elif expr.variables:
                s += self._print(expr.variables[0])
            s += r'\rightarrow '
            if len(expr.point) > 1:
                s += self._print(expr.point)
            else:
                s += self._print(expr.point[0])
        return r"O\left(%s\right)" % s
```
### 99 - sympy/printing/latex.py:

Start line: 1744, End line: 1752

```python
class LatexPrinter(Printer):

    def _print_DiracDelta(self, expr, exp=None):
        if len(expr.args) == 1 or expr.args[1] == 0:
            tex = r"\delta\left(%s\right)" % self._print(expr.args[0])
        else:
            tex = r"\delta^{\left( %s \right)}\left( %s \right)" % (
                self._print(expr.args[1]), self._print(expr.args[0]))
        if exp:
            tex = r"\left(%s\right)^{%s}" % (tex, exp)
        return tex
```
### 107 - sympy/printing/latex.py:

Start line: 1506, End line: 1527

```python
class LatexPrinter(Printer):

    def _print_BlockMatrix(self, expr):
        return self._print(expr.blocks)

    def _print_Transpose(self, expr):
        mat = expr.arg
        from sympy.matrices import MatrixSymbol
        if not isinstance(mat, MatrixSymbol):
            return r"\left(%s\right)^{T}" % self._print(mat)
        else:
            return "%s^{T}" % self._print(mat)

    def _print_Trace(self, expr):
        mat = expr.arg
        return r"\operatorname{tr}\left(%s \right)" % self._print(mat)

    def _print_Adjoint(self, expr):
        mat = expr.arg
        from sympy.matrices import MatrixSymbol
        if not isinstance(mat, MatrixSymbol):
            return r"\left(%s\right)^{\dagger}" % self._print(mat)
        else:
            return r"%s^{\dagger}" % self._print(mat)
```
### 108 - sympy/printing/latex.py:

Start line: 736, End line: 751

```python
class LatexPrinter(Printer):

    def _hprint_Function(self, func):
        r'''
        Logic to decide how to render a function to latex
          - if it is a recognized latex name, use the appropriate latex command
          - if it is a single letter, just use that letter
          - if it is a longer name, then put \operatorname{} around it and be
            mindful of undercores in the name
        '''
        func = self._deal_with_super_sub(func)
        if func in accepted_latex_functions:
            name = r"\%s" % func
        elif len(func) == 1 or func.startswith('\\'):
            name = func
        else:
            name = r"\operatorname{%s}" % func
        return name
```
### 109 - sympy/printing/latex.py:

Start line: 1766, End line: 1775

```python
class LatexPrinter(Printer):

    def _print_KroneckerDelta(self, expr, exp=None):
        i = self._print(expr.args[0])
        j = self._print(expr.args[1])
        if expr.args[0].is_Atom and expr.args[1].is_Atom:
            tex = r'\delta_{%s %s}' % (i, j)
        else:
            tex = r'\delta_{%s, %s}' % (i, j)
        if exp is not None:
            tex = r'\left(%s\right)^{%s}' % (tex, exp)
        return tex
```
### 110 - sympy/printing/latex.py:

Start line: 2290, End line: 2300

```python
class LatexPrinter(Printer):

    def _print_totient(self, expr, exp=None):
        if exp is not None:
            return r'\left(\phi\left(%s\right)\right)^{%s}' % \
                (self._print(expr.args[0]), self._print(exp))
        return r'\phi\left(%s\right)' % self._print(expr.args[0])

    def _print_reduced_totient(self, expr, exp=None):
        if exp is not None:
            return r'\left(\lambda\left(%s\right)\right)^{%s}' % \
                (self._print(expr.args[0]), self._print(exp))
        return r'\lambda\left(%s\right)' % self._print(expr.args[0])
```
### 111 - sympy/printing/latex.py:

Start line: 1425, End line: 1443

```python
class LatexPrinter(Printer):

    def _print_Relational(self, expr):
        if self._settings['itex']:
            gt = r"\gt"
            lt = r"\lt"
        else:
            gt = ">"
            lt = "<"

        charmap = {
            "==": "=",
            ">": gt,
            "<": lt,
            ">=": r"\geq",
            "<=": r"\leq",
            "!=": r"\neq",
        }

        return "%s %s %s" % (self._print(expr.lhs),
                             charmap[expr.rel_op], self._print(expr.rhs))
```
### 112 - sympy/printing/latex.py:

Start line: 722, End line: 734

```python
class LatexPrinter(Printer):

    def _print_Limit(self, expr):
        e, z, z0, dir = expr.args

        tex = r"\lim_{%s \to " % self._print(z)
        if str(dir) == '+-' or z0 in (S.Infinity, S.NegativeInfinity):
            tex += r"%s}" % self._print(z0)
        else:
            tex += r"%s^%s}" % (self._print(z0), self._print(dir))

        if isinstance(e, AssocOp):
            return r"%s\left(%s\right)" % (tex, self._print(e))
        else:
            return r"%s %s" % (tex, self._print(e))
```
### 122 - sympy/printing/latex.py:

Start line: 1147, End line: 1162

```python
class LatexPrinter(Printer):

    def _hprint_BesselBase(self, expr, exp, sym):
        tex = r"%s" % (sym)

        need_exp = False
        if exp is not None:
            if tex.find('^') == -1:
                tex = r"%s^{%s}" % (tex, self._print(exp))
            else:
                need_exp = True

        tex = r"%s_{%s}\left(%s\right)" % (tex, self._print(expr.order),
                                           self._print(expr.argument))

        if need_exp:
            tex = self._do_exponent(tex, exp)
        return tex
```
### 125 - sympy/printing/latex.py:

Start line: 932, End line: 945

```python
class LatexPrinter(Printer):

    def _print_LogOp(self, args, char):
        arg = args[0]
        if arg.is_Boolean and not arg.is_Not:
            tex = r"\left(%s\right)" % self._print(arg)
        else:
            tex = r"%s" % self._print(arg)

        for arg in args[1:]:
            if arg.is_Boolean and not arg.is_Not:
                tex += r" %s \left(%s\right)" % (char, self._print(arg))
            else:
                tex += r" %s %s" % (char, self._print(arg))

        return tex
```
### 127 - sympy/printing/latex.py:

Start line: 2322, End line: 2332

```python
class LatexPrinter(Printer):

    def _print_primenu(self, expr, exp=None):
        if exp is not None:
            return r'\left(\nu\left(%s\right)\right)^{%s}' % \
                (self._print(expr.args[0]), self._print(exp))
        return r'\nu\left(%s\right)' % self._print(expr.args[0])

    def _print_primeomega(self, expr, exp=None):
        if exp is not None:
            return r'\left(\Omega\left(%s\right)\right)^{%s}' % \
                (self._print(expr.args[0]), self._print(exp))
        return r'\Omega\left(%s\right)' % self._print(expr.args[0])
```
### 128 - sympy/printing/latex.py:

Start line: 1445, End line: 1456

```python
class LatexPrinter(Printer):

    def _print_Piecewise(self, expr):
        ecpairs = [r"%s & \text{for}\: %s" % (self._print(e), self._print(c))
                   for e, c in expr.args[:-1]]
        if expr.args[-1].cond == true:
            ecpairs.append(r"%s & \text{otherwise}" %
                           self._print(expr.args[-1].expr))
        else:
            ecpairs.append(r"%s & \text{for}\: %s" %
                           (self._print(expr.args[-1].expr),
                            self._print(expr.args[-1].cond)))
        tex = r"\begin{cases} %s \end{cases}"
        return tex % r" \\".join(ecpairs)
```
### 129 - sympy/printing/latex.py:

Start line: 574, End line: 594

```python
class LatexPrinter(Printer):

    def _print_UnevaluatedExpr(self, expr):
        return self._print(expr.args[0])

    def _print_Sum(self, expr):
        if len(expr.limits) == 1:
            tex = r"\sum_{%s=%s}^{%s} " % \
                tuple([self._print(i) for i in expr.limits[0]])
        else:
            def _format_ineq(l):
                return r"%s \leq %s \leq %s" % \
                    tuple([self._print(s) for s in (l[1], l[0], l[2])])

            tex = r"\sum_{\substack{%s}} " % \
                str.join('\\\\', [_format_ineq(l) for l in expr.limits])

        if isinstance(expr.function, Add):
            tex += r"\left(%s\right)" % self._print(expr.function)
        else:
            tex += self._print(expr.function)

        return tex
```
### 134 - sympy/printing/latex.py:

Start line: 1645, End line: 1676

```python
class LatexPrinter(Printer):

    _print_ImmutableDenseNDimArray = _print_NDimArray
    _print_ImmutableSparseNDimArray = _print_NDimArray
    _print_MutableDenseNDimArray = _print_NDimArray
    _print_MutableSparseNDimArray = _print_NDimArray

    def _printer_tensor_indices(self, name, indices, index_map={}):
        out_str = self._print(name)
        last_valence = None
        prev_map = None
        for index in indices:
            new_valence = index.is_up
            if ((index in index_map) or prev_map) and \
                    last_valence == new_valence:
                out_str += ","
            if last_valence != new_valence:
                if last_valence is not None:
                    out_str += "}"
                if index.is_up:
                    out_str += "{}^{"
                else:
                    out_str += "{}_{"
            out_str += self._print(index.args[0])
            if index in index_map:
                out_str += "="
                out_str += self._print(index_map[index])
                prev_map = True
            else:
                prev_map = False
            last_valence = new_valence
        if last_valence is not None:
            out_str += "}"
        return out_str
```
### 142 - sympy/printing/latex.py:

Start line: 615, End line: 643

```python
class LatexPrinter(Printer):

    def _print_BasisDependent(self, expr):
        from sympy.vector import Vector

        o1 = []
        if expr == expr.zero:
            return expr.zero._latex_form
        if isinstance(expr, Vector):
            items = expr.separate().items()
        else:
            items = [(0, expr)]

        for system, vect in items:
            inneritems = list(vect.components.items())
            inneritems.sort(key=lambda x: x[0].__str__())
            for k, v in inneritems:
                if v == 1:
                    o1.append(' + ' + k._latex_form)
                elif v == -1:
                    o1.append(' - ' + k._latex_form)
                else:
                    arg_str = '(' + LatexPrinter().doprint(v) + ')'
                    o1.append(' + ' + arg_str + k._latex_form)

        outstr = (''.join(o1))
        if outstr[1] != '-':
            outstr = outstr[3:]
        else:
            outstr = outstr[1:]
        return outstr
```
### 143 - sympy/printing/latex.py:

Start line: 348, End line: 364

```python
class LatexPrinter(Printer):

    def _print_Cycle(self, expr):
        from sympy.combinatorics.permutations import Permutation
        if expr.size == 0:
            return r"\left( \right)"
        expr = Permutation(expr)
        expr_perm = expr.cyclic_form
        siz = expr.size
        if expr.array_form[-1] == siz - 1:
            expr_perm = expr_perm + [[siz - 1]]
        term_tex = ''
        for i in expr_perm:
            term_tex += str(i).replace(',', r"\;")
        term_tex = term_tex.replace('[', r"\left( ")
        term_tex = term_tex.replace(']', r"\right)")
        return term_tex

    _print_Permutation = _print_Cycle
```
### 145 - sympy/printing/latex.py:

Start line: 1004, End line: 1013

```python
class LatexPrinter(Printer):

    def _print_elliptic_e(self, expr, exp=None):
        if len(expr.args) == 2:
            tex = r"\left(%s\middle| %s\right)" % \
                (self._print(expr.args[0]), self._print(expr.args[1]))
        else:
            tex = r"\left(%s\right)" % self._print(expr.args[0])
        if exp is not None:
            return r"E^{%s}%s" % (exp, tex)
        else:
            return r"E%s" % tex
```
### 146 - sympy/printing/latex.py:

Start line: 2312, End line: 2320

```python
class LatexPrinter(Printer):

    def _print_udivisor_sigma(self, expr, exp=None):
        if len(expr.args) == 2:
            tex = r"_%s\left(%s\right)" % tuple(map(self._print,
                                                (expr.args[1], expr.args[0])))
        else:
            tex = r"\left(%s\right)" % self._print(expr.args[0])
        if exp is not None:
            return r"\sigma^*^{%s}%s" % (self._print(exp), tex)
        return r"\sigma^*%s" % tex
```
### 149 - sympy/printing/latex.py:

Start line: 2063, End line: 2073

```python
class LatexPrinter(Printer):

    def _print_ComplexRootOf(self, root):
        cls = root.__class__.__name__
        if cls == "ComplexRootOf":
            cls = "CRootOf"
        expr = self._print(root.expr)
        index = root.index
        if cls in accepted_latex_functions:
            return r"\%s {\left(%s, %d\right)}" % (cls, expr, index)
        else:
            return r"\operatorname{%s} {\left(%s, %d\right)}" % (cls, expr,
                                                                 index)
```
### 150 - sympy/printing/latex.py:

Start line: 2302, End line: 2310

```python
class LatexPrinter(Printer):

    def _print_divisor_sigma(self, expr, exp=None):
        if len(expr.args) == 2:
            tex = r"_%s\left(%s\right)" % tuple(map(self._print,
                                                (expr.args[1], expr.args[0])))
        else:
            tex = r"\left(%s\right)" % self._print(expr.args[0])
        if exp is not None:
            return r"\sigma^{%s}%s" % (self._print(exp), tex)
        return r"\sigma%s" % tex
```
### 151 - sympy/printing/latex.py:

Start line: 1945, End line: 1955

```python
class LatexPrinter(Printer):

    def _print_ConditionSet(self, s):
        vars_print = ', '.join([self._print(var) for var in Tuple(s.sym)])
        if s.base_set is S.UniversalSet:
            return r"\left\{%s \mid %s \right\}" % \
                (vars_print, self._print(s.condition.as_expr()))

        return r"\left\{%s \mid %s \in %s \wedge %s \right\}" % (
            vars_print,
            vars_print,
            self._print(s.base_set),
            self._print(s.condition))
```
### 155 - sympy/printing/latex.py:

Start line: 2192, End line: 2220

```python
class LatexPrinter(Printer):

    def _print_Diagram(self, diagram):
        if not diagram.premises:
            # This is an empty diagram.
            return self._print(S.EmptySet)

        latex_result = self._print(diagram.premises)
        if diagram.conclusions:
            latex_result += "\\Longrightarrow %s" % \
                            self._print(diagram.conclusions)

        return latex_result

    def _print_DiagramGrid(self, grid):
        latex_result = "\\begin{array}{%s}\n" % ("c" * grid.width)

        for i in range(grid.height):
            for j in range(grid.width):
                if grid[i, j]:
                    latex_result += latex(grid[i, j])
                latex_result += " "
                if j != grid.width - 1:
                    latex_result += "& "

            if i != grid.height - 1:
                latex_result += "\\\\"
            latex_result += "\n"

        latex_result += "\\end{array}\n"
        return latex_result
```
### 156 - sympy/printing/latex.py:

Start line: 84, End line: 119

```python
modifier_dict = {
    # Accents
    'mathring': lambda s: r'\mathring{'+s+r'}',
    'ddddot': lambda s: r'\ddddot{'+s+r'}',
    'dddot': lambda s: r'\dddot{'+s+r'}',
    'ddot': lambda s: r'\ddot{'+s+r'}',
    'dot': lambda s: r'\dot{'+s+r'}',
    'check': lambda s: r'\check{'+s+r'}',
    'breve': lambda s: r'\breve{'+s+r'}',
    'acute': lambda s: r'\acute{'+s+r'}',
    'grave': lambda s: r'\grave{'+s+r'}',
    'tilde': lambda s: r'\tilde{'+s+r'}',
    'hat': lambda s: r'\hat{'+s+r'}',
    'bar': lambda s: r'\bar{'+s+r'}',
    'vec': lambda s: r'\vec{'+s+r'}',
    'prime': lambda s: "{"+s+"}'",
    'prm': lambda s: "{"+s+"}'",
    # Faces
    'bold': lambda s: r'\boldsymbol{'+s+r'}',
    'bm': lambda s: r'\boldsymbol{'+s+r'}',
    'cal': lambda s: r'\mathcal{'+s+r'}',
    'scr': lambda s: r'\mathscr{'+s+r'}',
    'frak': lambda s: r'\mathfrak{'+s+r'}',
    # Brackets
    'norm': lambda s: r'\left\|{'+s+r'}\right\|',
    'avg': lambda s: r'\left\langle{'+s+r'}\right\rangle',
    'abs': lambda s: r'\left|{'+s+r'}\right|',
    'mag': lambda s: r'\left|{'+s+r'}\right|',
}

greek_letters_set = frozenset(greeks)

_between_two_numbers_p = (
    re.compile(r'[0-9][} ]*$'),  # search
    re.compile(r'[{ ]*[-+0-9]'),  # match
)
```
### 157 - sympy/printing/latex.py:

Start line: 2075, End line: 2086

```python
class LatexPrinter(Printer):

    def _print_RootSum(self, expr):
        cls = expr.__class__.__name__
        args = [self._print(expr.expr)]

        if expr.fun is not S.IdentityFunction:
            args.append(self._print(expr.fun))

        if cls in accepted_latex_functions:
            return r"\%s {\left(%s\right)}" % (cls, ", ".join(args))
        else:
            return r"\operatorname{%s} {\left(%s\right)}" % (cls,
                                                             ", ".join(args))
```
### 167 - sympy/printing/latex.py:

Start line: 1015, End line: 1026

```python
class LatexPrinter(Printer):

    def _print_elliptic_pi(self, expr, exp=None):
        if len(expr.args) == 3:
            tex = r"\left(%s; %s\middle| %s\right)" % \
                (self._print(expr.args[0]), self._print(expr.args[1]),
                 self._print(expr.args[2]))
        else:
            tex = r"\left(%s\middle| %s\right)" % \
                (self._print(expr.args[0]), self._print(expr.args[1]))
        if exp is not None:
            return r"\Pi^{%s}%s" % (exp, tex)
        else:
            return r"\Pi%s" % tex
```
### 184 - sympy/printing/latex.py:

Start line: 1859, End line: 1879

```python
class LatexPrinter(Printer):

    def _print_SeqFormula(self, s):
        if len(s.start.free_symbols) > 0 or len(s.stop.free_symbols) > 0:
            return r"\left\{%s\right\}_{%s=%s}^{%s}" % (
                self._print(s.formula),
                self._print(s.variables[0]),
                self._print(s.start),
                self._print(s.stop)
            )
        if s.start is S.NegativeInfinity:
            stop = s.stop
            printset = (r'\ldots', s.coeff(stop - 3), s.coeff(stop - 2),
                        s.coeff(stop - 1), s.coeff(stop))
        elif s.stop is S.Infinity or s.length > 4:
            printset = s[:4]
            printset.append(r'\ldots')
        else:
            printset = tuple(s)

        return (r"\left[" +
                r", ".join(self._print(el) for el in printset) +
                r"\right]")
```
### 185 - sympy/printing/latex.py:

Start line: 678, End line: 686

```python
class LatexPrinter(Printer):

    def _print_Subs(self, subs):
        expr, old, new = subs.args
        latex_expr = self._print(expr)
        latex_old = (self._print(e) for e in old)
        latex_new = (self._print(e) for e in new)
        latex_subs = r'\\ '.join(
            e[0] + '=' + e[1] for e in zip(latex_old, latex_new))
        return r'\left. %s \right|_{\substack{ %s }}' % (latex_expr,
                                                         latex_subs)
```
### 195 - sympy/printing/latex.py:

Start line: 2543, End line: 2573

```python
def latex(expr, fold_frac_powers=False, fold_func_brackets=False,
          fold_short_frac=None, inv_trig_style="abbreviated",
          itex=False, ln_notation=False, long_frac_ratio=None,
          mat_delim="[", mat_str=None, mode="plain", mul_symbol=None,
          order=None, symbol_names=None, root_notation=True,
          mat_symbol_style="plain", imaginary_unit="i", gothic_re_im=False):
    if symbol_names is None:
        symbol_names = {}

    settings = {
        'fold_frac_powers': fold_frac_powers,
        'fold_func_brackets': fold_func_brackets,
        'fold_short_frac': fold_short_frac,
        'inv_trig_style': inv_trig_style,
        'itex': itex,
        'ln_notation': ln_notation,
        'long_frac_ratio': long_frac_ratio,
        'mat_delim': mat_delim,
        'mat_str': mat_str,
        'mode': mode,
        'mul_symbol': mul_symbol,
        'order': order,
        'symbol_names': symbol_names,
        'root_notation': root_notation,
        'mat_symbol_style': mat_symbol_style,
        'imaginary_unit': imaginary_unit,
        'gothic_re_im': gothic_re_im,
    }

    return LatexPrinter(settings).doprint(expr)


def print_latex(expr, **settings):
    """Prints LaTeX representation of the given expression. Takes the same
    settings as ``latex()``."""
    print(latex(expr, **settings))
```
### 212 - sympy/printing/latex.py:

Start line: 208, End line: 229

```python
class LatexPrinter(Printer):

    def doprint(self, expr):
        tex = Printer.doprint(self, expr)

        if self._settings['mode'] == 'plain':
            return tex
        elif self._settings['mode'] == 'inline':
            return r"$%s$" % tex
        elif self._settings['itex']:
            return r"$$%s$$" % tex
        else:
            env_str = self._settings['mode']
            return r"\begin{%s}%s\end{%s}" % (env_str, tex, env_str)

    def _needs_brackets(self, expr):
        """
        Returns True if the expression needs to be wrapped in brackets when
        printed, False otherwise. For example: a + b => True; a => False;
        10 => False; -10 => True.
        """
        return not ((expr.is_Integer and expr.is_nonnegative)
                    or (expr.is_Atom and (expr is not S.NegativeOne
                                          and expr.is_Rational is False)))
```