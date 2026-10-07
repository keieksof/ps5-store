# Find and download a game

Orbit includes 602 games with single-file options from Archive.org, Vikingfile and Fileditch. Each game appears once; open it to choose from its available sources and formats. The selection depends on which sources you enable.

## Browse and choose

**Discover opens first in 0.8.0.** Explore the featured game and Latest releases, then move down to All games. Its grid shows 48 games per page, ordered by the largest available download option first. Use Previous and Next to move between pages.

![Browser Discover All games grid](../assets/0.8.0/desktop-discover-grid.jpg)

Switch to **Browse** when you want search and filters. It defaults to **Release date (newest first)**. Games without a recorded date follow alphabetically. Search by game name or title ID, filter by source, format or download size, and change the sort order. **Reset filters** restores the default. Missing dates and other metadata can be added later through catalogue updates.

![Browse with filters and newest-first sorting](../assets/0.8.0/desktop-browse.jpg)

*Use filters to find options that fit your drive and preferred format. Sizes reflect the download option you choose.*

Save a game from its details page to find it under **Favourites** later; favourites are shared with your paired devices.

In the native TV app, use D-pad/left stick to move, Cross to select, Circle to go back and L1/R1 to switch tabs. Select filters with Cross; choose an option with the D-pad and Cross. The controls below apply to the browser version.

| Input | Action |
|---|---|
| D-pad or arrow keys | Move focus |
| Cross or Enter | Select the focused control |
| Circle or Escape | Go back or close a panel |
| Search: Down or Enter | Move into results |
| Search: Up | Return to the Browse tab |
| Search: Left or Right | Edit the text normally |
| Touch or mouse | Select a control |

## Pick a source, format and drive

On the game page, compare the options under **Download options**. Select the source and format you want, then use **Save to** to choose storage attached to the PS5. Check **Free now**, **Unfinished downloads**, and **After queue + this download** before starting.

![One game with Archive.org and Vikingfile choices](../assets/0.8.0/desktop-details.jpg)

*Different sources can offer different formats or sizes for the same game. Select the exact option you intend to download.*

For a direct option, select **Download to PS5**. Orbit adds it to Downloads after its preflight checks.

## Optional: download through TorBox

**Available in 0.8.0.** TorBox is an optional delivery choice for supported links. Update both Orbit’s service ELF and native FFPKG to use it in the TV app. You need your own TorBox account; its plan, host availability, quotas and file-size limits apply. You can keep using the original provider without connecting TorBox.

1. In Orbit’s browser version, open **App settings → Debrid**. You can do this on the PS5 or a paired phone or computer.
2. Get your API key from [TorBox settings](https://torbox.app/settings), enter it in Orbit and choose **Connect TorBox**. Keep the key private.
3. Open a game in either Orbit app and select its source and format. Under **Download using**, choose **TorBox**, then choose your destination drive.
4. Select **Download via TorBox**. Follow preparation and transfer progress in Downloads, where the job is labelled **via TorBox**.

![Browser Debrid settings with a sample connected TorBox account and an empty API-key field](../assets/0.8.0/desktop-torbox-setup.jpg)

*Connect the account in the browser once. The native TV app and paired devices share that connection.*

![Game details with TorBox selected and a PS5 destination drive](../assets/0.8.0/desktop-torbox-details.jpg)

An account connection does not guarantee support for every file. If an option is unavailable through TorBox, select the original provider explicitly. Orbit does not silently switch a TorBox job to another delivery method.

Pause and resume from Downloads as usual. Disconnecting TorBox pauses its unfinished jobs and keeps their partial files. Reconnect the same account to resume them. Disconnecting does not remove files from your remote TorBox account.

## Vikingfile and Fileditch: open the page on PS5 first

**Starting in the native TV app?** Choose **Open browser version** for a browser-only option. The browser opens your selected game with the same source and destination. Check those choices, then follow the steps below. If the source or drive is no longer available, select an available one explicitly. This does not start the provider step automatically. After Orbit queues the download, you can return to the TV app to follow it.

**Requires Orbit 0.5.0 or later** for Vikingfile. Fileditch options arrive through catalogue updates and need a Fileditch-enabled build. Older versions keep their Archive-only catalogue and do not receive unsupported options.

1. Select the provider option (Vikingfile or Fileditch) and destination drive in Orbit.
2. Select **Open download page on PS5**. This is the first step; it opens the provider page on the console.
3. Complete any verification yourself and select the provider’s **Download** button. Follow the file’s download controls if a redirect opens another provider page.
4. Return to Orbit Store and open **Downloads**. Orbit checks that the captured file matches the selected option before adding it to your queue.

![Vikingfile option and the three steps shown in Orbit](../assets/0.8.0/desktop-viking.jpg)

*The Open download page on PS5 button starts the provider step. Download on the provider page comes next; then return to Orbit.*

You can start the session from a paired phone, but the provider page and verification still appear on the PS5. There is no need to paste a generated link. If the session expires or reports no matching file, return to Orbit and start that option again. Only one browser verification session can run at a time; **Cancel verification** stops the pending session without adding a download.

Some previously checked Vikingfile options also offer **Download directly**. Fileditch options are file-page-only and always use the browser steps above.

<table>
  <tr><th>Choose an option on your phone</th><th>Start the PS5 browser step</th></tr>
  <tr>
    <td valign="top" width="50%"><img src="../assets/0.8.0/phone-details.jpg" width="280" alt="Phone game page with source and format options"></td>
    <td valign="top" width="50%"><img src="../assets/0.8.0/phone-viking.jpg" width="280" alt="Phone Vikingfile instructions and Open download page on PS5 button"></td>
  </tr>
</table>

*Screenshots are local previews of the 0.8.0 interface, using actual catalogue artwork with sample storage, provider availability and paired-console responses. The API-key field is empty. The actual Vikingfile provider page is operated on the PS5.*

## Fileditch Folder downloads install themselves

Fileditch options are single archive files marked **Folder**. After the download finishes and passes its checks, Orbit extracts the archive in the background — the queue shows **Extracting** — then moves the game folder into `homebrew` on the selected drive and removes the archive. There are no archive parts to collect and nothing to unpack by hand.

If Orbit stops during extraction, the download returns to **Paused**; resume it and extraction continues without downloading the file again. The archive is deleted only after a successful extraction, so a failed or cancelled extraction never loses the download.

## Follow your queue

Use **Active**, **Finished**, **Failed**, and **Cancelled** to find a transfer. Move waiting items up or down, pause and resume, or cancel. Cancelled items have their own view; Finished contains successfully completed downloads.

![Native Downloads showing paused and queued sample transfers labelled via TorBox](../assets/0.8.0/native-downloads.png)

<p><a href="../assets/0.8.0/phone-downloads.jpg"><img src="../assets/0.8.0/phone-downloads.jpg" width="320" alt="Phone Downloads showing the same sample TorBox queue"></a></p>

*The TV app and paired phone show the same queue, illustrated here with sample paused and queued jobs.*

New supported large downloads use four connections. Existing partials retain their original layout. Speed depends on the provider, network and drive. Pause, Resume and Cancel control the whole file.

After deleting a cancelled partial, you can start the download again from its game page. If you kept the partial, resume it from Downloads.

**Remove from history** and the history-clear buttons keep completed files. A cancelled item with a kept partial remains recoverable. To remove that partial, reconnect its original drive, select **Partial file options → Delete partial file**, then remove its history entry.

Keep the PS5 awake. Closing the TV app or control browser leaves the download worker running; stopping Orbit interrupts it. Interrupted transfers return paused after restart. Fileditch **Folder** downloads are extracted automatically after the download; otherwise Orbit does not directly install game packages, launch games or continue downloading in rest mode. For compatible drive files, see [Library](library.md).

## Get new games and corrected metadata

Orbit checks its catalogue on startup and every six hours. Use **App settings → Game catalogue → Refresh catalogue** for a manual check. The saved catalogue remains available offline, and queued downloads keep their original file identity. Catalogue additions and date corrections do not need an ELF update.

[Getting started](getting-started.md) · [Troubleshooting](troubleshooting.md) · [Project home](../README.md)
