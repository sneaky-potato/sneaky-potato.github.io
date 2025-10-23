---
title: "Quines"
date: 2024-06-18
description: "art"
tags: ["tech"]
---

{{< lead >}}
When programming becomes a piece of art.
{{< /lead >}}

## What is it?

A simple quine is its simplicity is a computer program that takes no input and produces a copy of its own source code as its only output. Think of it as a self replicating program. People have been coding quines for a long time now and have even gone ot the extent of creating more creative and smallest quines possible.
Here is a simple to understand quine written in C:

```C
#include <stdio.h>
#include <string.h>

int main() {
    const char* src = "#include <stdio.h>\n#include <string.h>\n\nint main() {\n    const char* src = \"?\";\n    size_t src_len = strlen(src);\n    for (size_t i=0; i<src_len; i++) {\n        if (src[i] == 63) {\n            for (size_t j=0; j<src_len; j++) {\n                switch (src[j]) {\n                    case '\\n': printf(\"\\\\n\"); break;\n                    case '\"': printf(\"\\\\\\\"\"); break;\n                    case '\\\\': printf(\"\\\\\\\\\"); break;\n                    default: printf(\"%c\", src[j]);\n                }\n            }\n        } else {\n            printf(\"%c\", src[i]);\n        }\n    }\n    return 0;\n}\n";
    size_t src_len = strlen(src);
    for (size_t i=0; i<src_len; i++) {
        if (src[i] == 63) {
            for (size_t j=0; j<src_len; j++) {
                switch (src[j]) {
                    case '\n': printf("\\n"); break;
                    case '"': printf("\\\""); break;
                    case '\\': printf("\\\\"); break;
                    default: printf("%c", src[j]);
                }
            }
        } else {
            printf("%c", src[i]);
        }
    }
    return 0;
}
```

All credits to [Tsoding Daily](https://www.youtube.com/@TsodingDaily) who creates a quine clock but in C in [this](https://www.youtube.com/watch?v=plFwBqBYpcY) video.

## More quines

- [clock](https://aem1k.com/qlock/)
- [world](https://aem1k.com/world/)
- [madelbrotset](https://aem1k.com/mandelcode/)
- [fire](https://aem1k.com/fire/quine/)
- [digital writing clock](https://www.youtube.com/watch?v=pgAHW8OpcTY)
- [analog clock](https://www.youtube.com/watch?v=FmxcnXhvJlo)
- [fluid](https://www.youtube.com/watch?v=QMYfkOtYYlg)

