

## EpicSplitter

4 chunks

#### Split 1
157 tokens, line: 0 - 17

```python
"""Helpers for the parser module."""

from typing import Tuple, List, Any, Iterator, TYPE_CHECKING

from sqlfluff.core.string_helpers import curtail_string

if TYPE_CHECKING:
    from sqlfluff.core.parser.segments import BaseSegment  # pragma: no cover


def join_segments_raw(segments: Tuple["BaseSegment", ...]) -> str:
    """Make a string from the joined `raw` attributes of an iterable of segments."""
    return "".join(s.raw for s in segments)


def join_segments_raw_curtailed(segments: Tuple["BaseSegment", ...], length=20) -> str:
    """Make a string up to a certain length from an iterable of segments."""
    return curtail_string(join_segments_raw(segments), length=length)
```



#### Split 2
134 tokens, line: 20 - 34

```python
def check_still_complete(
    segments_in: Tuple["BaseSegment", ...],
    matched_segments: Tuple["BaseSegment", ...],
    unmatched_segments: Tuple["BaseSegment", ...],
) -> bool:
    """Check that the segments in are the same as the segments out."""
    initial_str = join_segments_raw(segments_in)
    current_str = join_segments_raw(matched_segments + unmatched_segments)
    if initial_str != current_str:  # pragma: no cover
        raise RuntimeError(
            "Dropped elements in sequence matching! {!r} != {!r}".format(
                initial_str, current_str
            )
        )
    return True
```



#### Split 3
188 tokens, line: 37 - 61

```python
def trim_non_code_segments(
    segments: Tuple["BaseSegment", ...]
) -> Tuple[
    Tuple["BaseSegment", ...], Tuple["BaseSegment", ...], Tuple["BaseSegment", ...]
]:
    """Take segments and split off surrounding non-code segments as appropriate.

    We use slices to avoid creating too many unnecessary tuples.
    """
    pre_idx = 0
    seg_len = len(segments)
    post_idx = seg_len

    if segments:
        seg_len = len(segments)

        # Trim the start
        while pre_idx < seg_len and not segments[pre_idx].is_code:
            pre_idx += 1

        # Trim the end
        while post_idx > pre_idx and not segments[post_idx - 1].is_code:
            post_idx -= 1

    return segments[:pre_idx], segments[pre_idx:post_idx], segments[post_idx:]
```



#### Split 4
147 tokens, line: 64 - 84

```python
def iter_indices(seq: List, val: Any) -> Iterator[int]:
    """Iterate all indices in a list that val occurs at.

    Args:
        seq (list): A list to look for indices in.
        val: What to look for.

    Yields:
        int: The index of val in seq.

    Examples:
        The function works like str.index() but iterates all
        the results rather than returning the first.

        >>> print([i for i in iter_indices([1, 0, 2, 3, 2], 2)])
        [2, 4]
    """
    for idx, el in enumerate(seq):
        if el == val:
            yield idx
```
