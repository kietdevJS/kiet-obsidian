Obsidian supports special blockquote-based “callouts” to visually highlight information.  
They use the syntax:

```
> [!TYPE] Optional Title
> Content goes here
```

Below is a full reference of all callout features and variations.

---

## 1. Basic Callout Syntax

```
> [!info]
> This is an info callout.
```

---

## 2. Callouts With Custom Titles

```
> [!tip] Custom Tip Title
> You can name the callout anything you want.
```

---

## 3. Collapsible Callouts

### **A. Collapsed by default**

```
> [!note]- Collapsed Title
> This content starts collapsed.
```

### **B. Expanded by default**

```
> [!note]+ Expanded Title
> This content starts expanded.
```

### **C. No explicit symbol (defaults to expanded)**

```
> [!note] Title
> Content visible by default.
```

---

## 4. Common Built-in Callout Types

You may use any of the following supported keywords (case-insensitive):

- `note`
    
- `abstract`, `summary`, `tldr`
    
- `info`
    
- `todo`
    
- `tip`, `hint`, `important`
    
- `success`, `check`, `done`
    
- `question`, `help`, `faq`
    
- `warning`, `caution`, `attention`
    
- `failure`, `fail`, `missing`
    
- `danger`, `error`
    
- `bug`
    
- `example`
    
- `quote`, `cite`
    

**Example:**

```
> [!warning]
> Careful with this section.
```

---

## 5. Nesting Callouts

You can place callouts inside each other by indenting:

```
> [!info] Main Callout
> Some content here.
>
> > [!tip] Nested Callout
> > Additional helpful info.
```

---

## 6. Multi-Paragraph Callouts

Just continue using `>`:

```
> [!example] Multi-Paragraph
> Paragraph one of the example.
>
> Paragraph two continues the same callout.
```

---

## 7. Lists, Code Blocks, and Other Elements Inside Callouts

### **Lists**

```
> [!note]
> - Item 1
> - Item 2
```

### **Code Blocks**

```
> [!info]
> ```
> console.log("Inside a callout");
> ```
```

### **Headings**

```
> [!tip]
> ### Subheading inside callout
> Works fine.
```

---

## 8. Custom (Unofficial) Callout Types

Obsidian will render _any_ type string — even unsupported ones — using the default callout styling unless the theme includes extra styles.

```
> [!dragonfire] Custom Type
> Themes may style this uniquely.
```

---

## 9. Turning Off Callout Rendering

Add a backslash before the opening bracket to show the raw text:

```
> \[!info] This will not render as a callout.
```

---

## 10. Callouts in Lists

```
- Step 1
  > [!tip]
  > Helpful information for this step.
```

---

## 11. Callouts With No Content

```
> [!info]
```

Obsidian still renders an empty box.

---

## 12. Callout Styling Depends on Theme

Some community themes:

- Add more callout types
    
- Add icons
    
- Change colors
    
- Support callout-level CSS classes via snippets
    

You can also fully customize callouts using CSS like:

```css
.callout[data-callout="warning"] {
  --callout-color: red;
}
```

---

# Summary Table

|Feature|Supported?|Example|
|---|---|---|
|Basic callout|✔️|`[!info]`|
|Custom title|✔️|`[!tip] My Title`|
|Collapsible|✔️|`[!note]-`|
|Nested callouts|✔️|Using `>>`|
|Lists, code blocks inside|✔️|Works|
|Custom callout names|✔️|`[!dragonfire]`|
|Escape callout|✔️|`\[!info]`|

---

If you want, I can also generate:

✅ A downloadable `.md` file  
✅ A full cheatsheet graphic  
✅ Examples for each callout type  
✅ A theme-optimized version for your personal Obsidian setup