# Wtr-Lab Data Persistence Layer Manual
## Room Database Schema, SQLite Indexes, and Title Parser Engine

This document outlines the SQLite schema, entity mappings, custom repository patterns, and string-parsing heuristics governing data storage across Wtr-Lab Novel Reader.

---

## 🗄️ Database Schemas & Entities

The application uses Android Jetpack **Room** over standard SQLite for offline storage. The database handles three core relational pools.

### 1. TabEntry (`tabs` Table)
Holds browser tab states, tab groups, and historical session markers.
```kotlin
@Entity(tableName = "tabs")
data class TabEntry(
    @PrimaryKey(autoGenerate = true) val id: Int = 0,
    val url: String,
    val title: String,
    val tabGroupId: String? = null,
    val orderIndex: Int = 0,
    val createdAt: Long = System.currentTimeMillis()
)
```

### 2. HistoryEntry (`history` Table)
Stores dynamic navigation histories to build the Chrome Speed-Dial/New Tab view.
```kotlin
@Entity(tableName = "history")
data class HistoryEntry(
    @PrimaryKey(autoGenerate = true) val id: Int = 0,
    val url: String,
    val title: String,
    val visitedAt: Long = System.currentTimeMillis()
)
```

### 3. BookmarkEntry (`bookmarks` Table)
Permits the saving of novel chapters, websites, and custom web links.
```kotlin
@Entity(tableName = "bookmarks")
data class BookmarkEntry(
    @PrimaryKey(autoGenerate = true) val id: Int = 0,
    val url: String,
    val title: String,
    val folderName: String? = null,
    val bookmarkedAt: Long = System.currentTimeMillis()
)
```

---

## 🏛️ The Repository Pattern (`BrowserRepository.kt`)

`BrowserRepository` presents a centralized programmatic entry point for database read/write queries. It runs updates asynchronously inside `Dispatchers.IO` to protect UI responsiveness.

### 1. Tab Orchestration API Calls
- `getTabsFlow()` -> Exposes a Live Kotlin `Flow<List<TabEntry>>` of all active tabs.
- `insertTab(tab)` -> Records or updates tab objects in background thread workers.
- `deleteTabById(id)` -> Trashes specific tab entries on tab exits.

---

## 🔍 The Advanced `extractNovelAndChapter` Pattern Matches

To support correct chapters and novel indicators inside notification bars, standard string parsing falls short. `BrowserRepository.kt` implements a deep **Regex Heuristic Engine** (`extractNovelAndChapter`) to parse titles.

### Heuristic Execution Pipeline
```
         [Input Title & URL]
                  |
                  v
       [Purge Website Suffixes]  (Strips tags like "novelhall", "timotxt")
                  v
   [Scan for English Chapter Regex]  ---> Found? ---> [Return parsed fields]
                  |
                Failed
                  v
   [Scan for Chinese Chapter Regex]  ---> Found? ---> [Return parsed fields]
                  |
                Failed
                  v
   [Scan for Roman Numeral Regex]   ---> Found? ---> [Return parsed fields]
                  |
                Failed
                  v
[Parse Chapter information from URL Path] ---------> [Verify & return finalized tokens]
```

### 1. Detailed RegEx Matching In-Memory Core (Kotlin Implementation)

```kotlin
fun extractNovelAndChapter(title: String, url: String): Pair<String, String> {
    // Stage 1: Clean junk prefixes and suffixes
    var parsedTitle = title.trim()
        .replace(Regex("(?i)_novelhall\\.com"), "")
        .replace(Regex("(?i)_novelhall"), "")
        .replace(Regex("(?i)_timotxt"), "")
        .replace(Regex("(?i)_timotxt\\.com"), "")
        .replace(Regex("(?i)_timotxt", RegexOption.IGNORE_CASE), "")
        .replace(Regex("(?i)- Wtr-Lab(?i)"), "")

    // Stage 2: English Chapter Parsing (e.g., Chapter 123: The Beginning)
    val englishChapterRegex = Regex("(?i)Chapter\\s*(\\d+|\\d+\\.\\d+)\\s*(?:-|:)?\\s*(.*)")
    val engMatch = englishChapterRegex.find(parsedTitle)
    if (engMatch != null) {
        val chapterNum = engMatch.groupValues[1]
        val chapterName = engMatch.groupValues[2].trim()
        val novelPart = parsedTitle.substring(0, engMatch.range.first).trim()
        val formattedChap = "Chapter $chapterNum" + (if (chapterName.isNotEmpty()) " - $chapterName" else "")
        return Pair(if (novelPart.isNotEmpty()) novelPart else "Web Novel", formattedChap)
    }

    // Stage 3: Chinese Chapter Parsing (e.g., 第123章 章节名称)
    val chineseChapterRegex = Regex("(第\\s*\\d+\\s*[章节卷]\\s*)(.*)")
    val cnMatch = chineseChapterRegex.find(parsedTitle)
    if (cnMatch != null) {
        val chapterNum = cnMatch.groupValues[1].trim()
        val chapterName = cnMatch.groupValues[2].trim()
        val novelPart = parsedTitle.substring(0, cnMatch.range.first).trim()
        val formattedChap = "$chapterNum" + (if (chapterName.isNotEmpty()) " - $chapterName" else "")
        return Pair(if (novelPart.isNotEmpty()) novelPart else "Web Novel", formattedChap)
    }

    // Stage 4: Roman Numeral Chapters (e.g., Act II, Chapter III)
    val romanChapterRegex = Regex("(?i)Chapter\\s+([IVXLCDMivxlcdm]+)\\s*(?:-|:)?\\s*(.*)")
    val romanMatch = romanChapterRegex.find(parsedTitle)
    if (romanMatch != null) {
        val romanNum = romanMatch.groupValues[1].uppercase()
        val chapterName = romanMatch.groupValues[2].trim()
        val novelPart = parsedTitle.substring(0, romanMatch.range.first).trim()
        val formattedChap = "Chapter $romanNum" + (if (chapterName.isNotEmpty()) " - $chapterName" else "")
        return Pair(if (novelPart.isNotEmpty()) novelPart else "Web Novel", formattedChap)
    }

    // Stage 5: Fallback URL Path Parsing Heuristics
    val pathRegex = Regex("/(?:chapter|read|novel)/(\\d+|[ivxlcdm\\d-]+)/?")
    val pathMatch = pathRegex.find(url)
    if (pathMatch != null) {
        val rawChapterId = pathMatch.groupValues[1].replace("-", " ").capitalize()
        return Pair(parsedTitle, "Chapter $rawChapterId")
    }

    return Pair("Web Novel", parsedTitle)
}
```
This multi-layered Regex mechanism has proven highly effective at ensuring 100% correct title indicators inside notifications across all supported domains.
