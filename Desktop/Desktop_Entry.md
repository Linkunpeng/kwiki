`.desktop` 文件是 Linux 桌面环境中用于描述应用程序启动方式、显示名称、图标等信息的**桌面项（Desktop Entry）文件**。由freedesktop.org发布的标准你。可以将它们理解为 Windows 系统中的快捷方式，但功能更强大，并且是跨桌面环境（如 GNOME、KDE、XFCE 等）的标准。

-----

## `.desktop` 文件是什么？

`.desktop` 文件是遵循 [Desktop Entry Specification](https://specifications.freedesktop.org/desktop-entry-spec/latest/) 标准的纯文本文件。它们告诉桌面环境如何：

  * **启动一个应用程序**：指定要执行的命令。
  * **显示应用程序信息**：包括应用程序的名称、通用名称、描述等。
  * **显示应用程序图标**：指定用于显示在菜单、面板或桌面上的图标。
  * **在哪个菜单类别中显示**：例如，“图形”、“开发”、“实用工具”等。
  * **支持哪些MIME类型**：当打开特定类型的文件时，系统可以使用此应用程序。
  * **提供快速操作**：通过右键菜单提供额外的功能（例如，在浏览器中打开一个新匿名窗口）。

-----

## `.desktop` 文件如何工作？

当桌面环境启动时，它会扫描特定的目录以查找 `.desktop` 文件。一旦找到，它会解析这些文件并使用其中的信息来构建应用程序菜单、显示桌面图标、以及处理文件关联等。

以下是 `.desktop` 文件工作的一些关键方面：

1.  **文件位置**：

      * **`/usr/share/applications/`**: 这是系统级的目录，存放由软件包安装的应用程序 `.desktop` 文件。这些文件通常对所有用户可见。
      * **`~/.local/share/applications/`**: 这是用户级的目录，存放用户自己创建或修改的 `.desktop` 文件。这里的 `.desktop` 文件会覆盖 `/usr/share/applications/` 中同名的文件，或添加新的用户自定义应用程序。
      * **`/usr/local/share/applications/`**: 存放本地安装但不是通过包管理器安装的应用程序的 `.desktop` 文件。

2.  **解析与显示**：桌面环境（如 GNOME Shell、KDE Plasma 的 `ksmserver` 和 `kicker` 组件等）会持续监视这些目录。当检测到新的或修改过的 `.desktop` 文件时，它们会重新解析并更新应用程序菜单或桌面图标。

3.  **执行命令**：当用户点击应用程序菜单中的图标或 `.desktop` 文件本身时，桌面环境会读取 `.desktop` 文件中的 `Exec` 字段，并执行其中指定的命令来启动应用程序。

4.  **国际化支持**：`.desktop` 文件支持多语言。你可以为 `Name`、`Comment` 等字段提供不同语言的版本，例如 `Name[zh_CN]=浏览器`。桌面环境会根据当前的系统语言设置显示相应的文本。

5.  **桌面操作（Desktop Actions）**：通过 `Actions` 字段和对应的 `[Desktop Action <name>]` 节，可以为应用程序定义额外的操作。这些操作通常在应用程序的右键菜单中显示，例如，一个文本编辑器可能提供“新建文件”或“打开最近文件”的快速操作。

-----

## `.desktop` 文件结构示例

一个典型的 `.desktop` 文件由多个节（Section）组成，每个节包含一系列键值对。

```ini
[Desktop Entry]
Version=1.0
Type=Application
Name=Firefox Web Browser
Comment=Browse the World Wide Web
Exec=/usr/bin/firefox %u
Icon=firefox
Terminal=false
Categories=Network;WebBrowser;
MimeType=text/html;text/xml;application/xhtml+xml;application/xml;application/vnd.mozilla.xul+xml;application/rss+xml;application/rdf+xml;image/gif;image/jpeg;image/png;image/webp;video/webm;
StartupNotify=true
Actions=new-window;new-private-window;

[Desktop Action new-window]
Name=New Window
Exec=/usr/bin/firefox --new-window
OnlyShowIn=Unity;GNOME;

[Desktop Action new-private-window]
Name=New Private Window
Exec=/usr/bin/firefox --private-window
OnlyShowIn=Unity;GNOME;
```

**关键字段解释：**

  * **`[Desktop Entry]`**: 所有的桌面项文件都必须以这个节开始。
  * **`Version`**: 规范版本。
  * **`Type`**: 桌面项的类型。最常见的是 `Application`（应用程序）、`Link`（链接到文件/URL）和 `Directory`（目录）。
  * **`Name`**: 应用程序在菜单中显示的名称。
  * **`Comment`**: 应用程序的简短描述或提示。
  * **`Exec`**: **最重要**的字段，指定启动应用程序时要执行的命令。
      * `%u`：会被一个或多个 URL 替换，常用于浏览器或文件管理器。
      * `%f`：会被一个本地文件路径替换。
      * `%F`：会被一个或多个本地文件路径替换。
  * **`Icon`**: 应用程序的图标名称或完整路径。如果只是名称，系统会在主题图标路径中查找。
  * **`Terminal`**: 如果为 `true`，则在终端中运行 `Exec` 命令；如果为 `false`，则直接运行。
  * **`Categories`**: 应用程序所属的类别，用于在应用程序菜单中进行分类。多个类别用分号 `;` 分隔。
  * **`MimeType`**: 应用程序能够打开的文件MIME类型列表，用分号 `;` 分隔。
  * **`StartupNotify`**: 如果为 `true`，桌面环境会在应用程序启动时显示启动反馈（例如，鼠标指针变为沙漏），直到应用程序准备就绪。
  * **`Actions`**: 列出该应用程序支持的桌面操作的名称。
  * **`[Desktop Action <name>]`**: 定义具体的桌面操作，包含 `Name`（操作名称）和 `Exec`（执行的命令）。
  * **`OnlyShowIn`**: 指定此操作只在哪些桌面环境中显示。

通过理解和利用 `.desktop` 文件，用户和开发者可以更好地集成应用程序到 Linux 桌面环境，并提供更丰富的用户体验。
