# arillo/silverstripe-simple-search

Adds simple site search to your project.

### Requirements

SilverStripe CMS ^4.0

### Simple page example

```php
<?php
use SilverStripe\CMS\Model\SiteTree;
use Arillo\SimpleSearch\ISearchIndexable;
use Arillo\SimpleSearch\SearchableExtension;
use Arillo\SimpleSearch\SearchIndexEntry;

class Page extends SiteTree implements ISearchIndexable // you need to implement this interface
{
    private static $extensions = [
        // adds after write/delete hooks to re-index this record
        SearchableExtension::class,
    ];

    // add interface function, it should return the string you want to add to your search index.
    public function forSearchIndex()
    {
        $contents = [
            $this->Title,
            $this->obj('Content')
                ->setProcessShortcodes(true)
                ->RAW()
        ];

        $string = implode($contents, ' ');
        return $string ? SearchIndexEntry::sanitize_string($string) : null;
    }
}

```

### (Re-)Build the search index

```
php vendor/silverstripe/framework/cli-script.php dev/tasks/Arillo-SimpleSearch-BuildIndexTask
```


### Integrate with Fluent

Add `search.yml` to your config:

```
Arillo\SimpleSearch\SearchIndexEntry:
  extensions:
    - TractorCow\Fluent\Extension\FluentExtension
  translate:
    - Title
    - SearchableText
```

### Integrate with arillo/elements

Add the extension, so update / publish elements will trigger an reindex of the holder page.

```yaml
Arillo\Elements\ElementBase:
  extensions:
    - Arillo\SimpleSearch\ElementDataExtension
```

Implement `ISearchIndexable` e.g. like this:

```php
<?php
use SilverStripe\CMS\Model\SiteTree;
use SilverStripe\View\SSViewer;
use SilverStripe\Core\Config\Config;
use Arillo\SimpleSearch\ISearchIndexable;
use Arillo\SimpleSearch\SearchableExtension;
use Arillo\SimpleSearch\SearchIndexEntry;
use Arillo\Elements\ElementsExtension;

class Page extends SiteTree implements ISearchIndexable // you need to implement this interface
{
	const SECTIONS = 'Sections';

    private static $extensions = [
        // adds after write/delete hooks to re-index this record
        SearchableExtension::class,
    ];

    // add interface function, it should return the string you want to add to your search index.
    public function forSearchIndex()
    {
	 if ($this->isPageWithSections()) {
            $oldThemes = SSViewer::get_themes();
            SSViewer::set_themes(
                Config::inst()->get(SSViewer::class, 'themes')
            );
            try {
                $string = SearchIndexEntry::sanitize_string(
                    $this->customise([
                        'RelationName' => self::SECTIONS
                    ])->renderWith('Elements')
                );
            } finally {
                SSViewer::set_themes($oldThemes);
            }
            return $string;
        }

        $contents = [
            $this->Title,
            $this->obj('Content')
                ->setProcessShortcodes(true)
                ->RAW()
        ];

        $string = implode($contents, ' ');
        return $string ? SearchIndexEntry::sanitize_string($string) : null;
    }
    
    public function isPageWithSections()
    {
        return isset(
            ElementsExtension::page_element_relation_names($this)[
                self::SECTIONS
            ]
        );
    }
}
```

### Content stripping

When using `SearchIndexEntry::sanitize_string` html tags will be removed from the text.
It is possible to mark certain parts of the html to also be removed. This can be handy if you want to strip out navigation elements or breadcrumbs. To accomplish this all content between `<!--<SearchExclude>-->` and `<!--</SearchExclude>-->` will be ereased:

```html
<!--<SearchExclude>-->
    Contents between this comments will be stripped
<!--</SearchExclude>-->
```
