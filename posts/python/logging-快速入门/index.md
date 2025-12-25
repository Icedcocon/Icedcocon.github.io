# logging 快速入门

```python
import logging
from logging.config import dictConfig

VLLM_LOGGING_PREFIX: str = ""
VLLM_LOGGING_LEVEL: str = "INFO"
VLLM_LOGGING_STREAM: str = "ext://sys.stdout"
_FORMAT = (f"{VLLM_LOGGING_PREFIX}%(levelname)s %(asctime)s "
           "[%(fileinfo)s:%(lineno)d] %(message)s")
_DATE_FORMAT = "%m-%d %H:%M:%S"
# 这个变量是Logger的名字，请务必更改
LOGGER_NAME:str = "logger"

DEFAULT_LOGGING_CONFIG = {
    "formatters": {
        "vllm": {
            "class": "vllm.logging_utils.NewLineFormatter",
            "datefmt": _DATE_FORMAT,
            "format": _FORMAT,
        },
    },
    "handlers": {
        "vllm": {
            "class": "logging.StreamHandler",
            "formatter": "vllm",
            "level": VLLM_LOGGING_LEVEL,
            "stream": VLLM_LOGGING_STREAM,
        },
    },
    "loggers": {
        LOGGER_NAME: {
            "handlers": ["vllm"],
            "level": "DEBUG",
            "propagate": False,
        },
    },
    "version": 1,
    "disable_existing_loggers": False
}



dictConfig(DEFAULT_LOGGING_CONFIG)
logger = logging.getLogger(LOGGER_NAME)
logger.info("hello world")
```

### formatters.vllm.class 

> [!NOTE]
> 需要放在独立的模块中，且确保可以导入


```python
import logging
from pathlib import Path

from vllm import envs


class NewLineFormatter(logging.Formatter):
    """Adds logging prefix to newlines to align multi-line messages."""

    def __init__(self, fmt, datefmt=None, style="%"):
        super().__init__(fmt, datefmt, style)

        self.use_relpath = envs.VLLM_LOGGING_LEVEL == "DEBUG"
        if self.use_relpath:
            self.root_dir = Path(__file__).resolve().parent.parent.parent

    def format(self, record):

        def shrink_path(relpath: Path) -> str:
            """
            Shortens a file path for logging display:
            - Removes leading 'vllm' folder if present.
            - If path starts with 'v1',
            keeps the first two and last two levels,
            collapsing the middle as '...'.
            - Otherwise, keeps the first and last two levels,
            collapsing the middle as '...'.
            - If the path is short, returns it as-is.
            - Examples:
            vllm/model_executor/layers/quantization/utils/fp8_utils.py ->
            model_executor/.../quantization/utils/fp8_utils.py
            vllm/model_executor/layers/quantization/awq.py ->
            model_executor/layers/quantization/awq.py
            vllm/v1/attention/backends/mla/common.py ->
            v1/attention/backends/mla/common.py

            Args:
                relpath (Path): The relative path to be shortened.
            Returns:
                str: The shortened path string for display.
            """
            parts = list(relpath.parts)
            new_parts = []
            if parts and parts[0] == "vllm":
                parts = parts[1:]
            if parts and parts[0] == "v1":
                new_parts += parts[:2]
                parts = parts[2:]
            elif parts:
                new_parts += parts[:1]
                parts = parts[1:]
            if len(parts) > 2:
                new_parts += ["..."] + parts[-2:]
            else:
                new_parts += parts
            return "/".join(new_parts)

        if self.use_relpath:
            abs_path = getattr(record, "pathname", None)
            if abs_path:
                try:
                    relpath = Path(abs_path).resolve().relative_to(
                        self.root_dir)
                except Exception:
                    relpath = Path(record.filename)
            else:
                relpath = Path(record.filename)
            record.fileinfo = shrink_path(relpath)
        else:
            record.fileinfo = record.filename

        msg = super().format(record)
        if record.message != "":
            parts = msg.split(record.message)
            msg = msg.replace("\n", "\r\n" + parts[0])
        return msg

```
