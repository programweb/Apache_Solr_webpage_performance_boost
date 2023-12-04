# Apache Solr Search with webpage performance boost
## Techniques used to deal with execution timeouts and high memory usage.

1. Change the ini server configurations on-the-fly just in the function ihme_search_export_build_csv
   * Usually you can get away with just putting these ini_set calls at the top of the file; however, since the search is on almost every page, doing that was equivalent to changing the ini server configs globally. Doing it globally would cause other problems; eg: if you have a page with an error it might take a long time to finish loading and resolve.
   * I experimented with a number of settings but found significantly increasing the **max execution time and memory limit** to be critical to address the bug in question.
The ini_set function conveniently returns the initial ini server configurations.

1. Use a PHP **registered shutdown function** to reset the ini server configurations to their initial values.
   * I tried to do just ini set again at the bottom of the function, but that resulted in a zero-byte CSV download
   * So, I had to use a registered shutdown function which fires when the script (not just the function) has completed.
   * Interestingly, you can pass the initial values into the shutdown function.

```php
/**
 * Shutdown function to reset ini configuration values 
 * after script exit of CSV Export download
 * to original settings prior to the export.
 */
function shutdown_search_result_export($initial_mem_limit, $initial_max_ex){
	ini_set('memory_limit', $initial_mem_limit);
	ini_set('max_execution_time',  $initial_max_ex);
}
```
&nbsp;

      + The function shutdown_search_result_export above is registered below with a call to: register_shutdown_function


3. I determined the node_load / and multi_node_load calls were the most consumptive of resources . A lot of database joins and queries go on in there, as well as building OOP objects (OOP is more taxing than procedural code in PHP)
   * Anne has a good idea of not using the Drupal core function node_load but rather using our own SQL creating more minimal objects/arrays.

4. Apache Solr produced a huge number of results for this bug
   * the webpages displaying the search results load fast because there is a start and offset set in the SQL; so, each page only shows a small portion of the total results
   * the CSV download, however, contains all results
      * which, incidentally, are much more as a logged-in IHME user than an anonymous user; This is known, perhaps desirable, expected and out-of-scope for this ticket

5. A **generator function** node_load_helper is used because it consumes less memory when looping through a large array of large objects. They are iterators under-the-hood. I have experience making generators, and I made a custom one here which is not documented anywhere that I know of. There is a loop in the generator, but it only yields one iteration per call. It yields an array â€” the variables are set in a list assignation (aka destructuring assignment) coupled with the generator call. PHP generators MUST be called with a foreach loop as is done here (not a while, for, do while, etc.).


```php

/**
 * Generator function for exports
 * generator to reduce memory consumption
 * avoiding timeouts in large "exports" (CSV downloads)
 * 
 * @param $results passed by reference to avoid an array copy
 * 
 */
function node_load_helper( &$results ){
	foreach($results as $result) {
		if($result['node']->bundle != 'record') continue;
		if( ! $result['node']->entity_id ) continue;
		$node = node_load($result['node']->entity_id);
		if( ! $node ) continue;
		yield [$result['node']->entity_id, $node];
	}
 }
```

6. The array/object is **passed in by reference** (the "&" sign in the function argument &$results) to the generator function otherwise a copy would be made with each call (taxing server memory).

7. Arrays no longer in use, eg 'results', are emptied when no longer needed–freeing some memory.

## The function below was adapted in  a number of ways (code changes made) for the above efforts to work.
```php
/**
 * Callback for search/site/export_csv/%
 */
function ihme_search_export_build_csv($q) {
	global $user;

	$initial_mem_limit = ini_set('memory_limit','4096M'); // memory limit increase required for large export (CSV download)
	$initial_max_ex = ini_set('max_execution_time', 3000); // max seconds required for large export (CSV download)
	register_shutdown_function('shutdown_search_result_export', $initial_mem_limit, $initial_max_ex);

	$keys = (urldecode($q)); // decode query or special characters will be ignored
	$filters = isset($_GET['f']) ? $_GET['f'] : '';
	$solrsort = isset($_GET['solrsort']) ? $_GET['solrsort'] : '';
	$page = isset($_GET['page']) ? $_GET['page'] : 0;

	// make search request
	try {
			$params['q'] = $keys;
			$params['rows'] = 2147483647; // largest number possible without overflowing
			if($filters) {
				$params['fq'] = $filters;
			}

			$results = apachesolr_search_run('search', $params, $solrsort, 'search/site/', pager_find_page());
	}
	catch (Exception $e) {
			watchdog('Apache Solr', nl2br(check_plain($e->getMessage())), NULL, WATCHDOG_ERROR);
	}

	// create nodes array from results
	$nodes = []; // array of node objects indexed by nid
	$output = '';
	$t =0;
	foreach( node_load_helper($results) as $node_and_id ){
		list($node_id, $node) = $node_and_id;
		$nodes[ $node_id ] = $node;
		$t++;
	}

	// free up some memory
	$params = [];
	$results = [];

	$access_fields = ihme_search_export_get_fields('record', current($nodes));
  	$access_fields = ihme_search_export_get_additional_field_attributes($access_fields);

	// set column headers
	// if logged in, return NIDs
	if($user->uid) {
		$output .= "NID,Title";
	}
	else {
		$output .= "Title";
	}

	foreach ($access_fields as $field){
		$field_label = field_info_instance('node', $field[0], 'record');
		$output .= ',' . $field_label['label'];
		if($field[0] == 'field_internal_files') {
			$output .= ',Filename,Status,Description,Location,Full Path,Hash';
		}
	}

	$output .= "\n";

	// set fields
	foreach($nodes as $node) {
		$field_references = NULL;
		// if logged in, return NIDs
		if($user->uid) {
			$items = array($node->nid,$node->title);
		}
		else {
			$items = array($node->title);
		}

		foreach($access_fields as $key => $value) {
			$field = $node->{$value[0]}['und'];
			switch($value[1]) {
				case 'taxonomy_term_reference':
					if($field) {
						$items[] = _ihme_search_export_parse_taxonomy($field);
					} else {
						$items[] = '';
					}
					break;
				case 'text':
					if($field) {
						$items[] = _ihme_search_export_parse_text($field);
					} else {
						$items[] = '';
					}
					break;
				case 'datetime':
					if($field) {
						$items[] = _ihme_search_export_parse_time($field, $value[2]);
					} else {
						$items[] = '';
					}
					break;
				case 'list_boolean':
					if($field) {
						$items[] = ($field[0]['value']) ? 'True' : 'False';
					} else {
						$items[] = '';
					}
					break;
				case 'entityreference':
					if($value[0] == 'field_internal_files') {
						$file_references = $field;
						break;
					} else {
						if($field) {
							$items[] = _ihme_search_export_parse_entity($value, $field);
						} else {
							$items[] = '';
						}
					}
					break;
				case 'list_text':
					if($field) {
						$items[] = $value['allowed_values'][$field[0]['value']];
					}
					else{
						$items[] = '';
					}
					break;
				case 'text_long':
					if($field) {
						$items[] = ($field[0]['value']) ? $field[0]['value'] : '';
					}
					else{
						$items[] = '';
					}
					break;
				default:
					$items[] = 'Error: Field not supported by ihme search export module';
					break;
			} // end switch
		}

		$items = array_map('_ihme_search_export_csv_escape', $items);
		$output .= implode($items, ",");
		$output .= "\n";

		// ASSOCIATED FILES
    	if ($file_references){
			foreach ($file_references as $key => $file){
				$fileitems = array();
				$filenode = node_load($file['target_id']);
 				$fileitems[] = 'file';
				$fileitems[] = $filenode->title;
				$fileitems[] = taxonomy_term_load($filenode->field_status['und']['0']['tid'])->name;
				$fileitems[] = $filenode->field_description['und']['0']['value'];
				$fileitems[] = _ihme_search_export_slashify($filenode->field_location['und']['0']['value']);
				$fileitems[] = _ihme_search_export_slashify($filenode->field_location['und']['0']['value'].'/'.$filenode->title);
				$fileitems[] = $filenode->field_hash['und']['0']['value'];
				//,filename,status,description,location,full path,hash
				$fileitems = array_map('_ihme_search_export_csv_escape', $fileitems);
				$output .= implode($items, ",").','.implode($fileitems, ",");
				$output .= "\n";
			}
    	}
	}

	drupal_add_http_header('Content-Type', 'text/csv; utf-8');
	drupal_add_http_header('Content-Disposition', 'attachment; filename="searchresults.csv');
	print $output;

}
```

