<?php

if (!class_exists('MA_CustomFonts')) :
class MA_CustomFonts {

	const TITLE     	= 'MA Custom Fonts';
	const VERSION   	= '3.2.1';

	// ===== CONFIGURATION =====
	public static $recursive 		= true; 	// enables recursive file scan
	public static $parsename 		= true; 	// enables parsing font weight and style from file name
	public static $fontdisplay		= 'block';	// set font-display to auto, block, swap, fallback, optional or '' (disable)
	public static $cssoutput		= 'file';	// set to 'html' to output CSS inline into page, 
												// set to 'file' to create and reference a CSS file (cacheable by browser)
	public static $cssminimize		= true; 	// minimize CSS (true) or pretty print (false)

	public static $timing			= false; 	// write timing info (a lot!) to wordpress debug.log if WP_DEBUG enabled		
	public static $debug			= false; 	// write debug info (a lot!) to wordpress debug.log if WP_DEBUG enabled	
	public static $sample_text 		= 'The quick brown fox jumps over the lazy dog.';	

	// ===== INTERNAL =====
	public 	static $prioritized_formats	= ['eot','woff2','woff','ttf','otf','svg'];
	private static $fonts 				= null;	// will be populated with fonts and related files we found
	private static $fonts_details_cache	= [];	// cache for already parsed font details
	private static $font_files_cnt		= 0;	// number of font files parsed
	
	//-------------------------------------------------------------------------------------------------------------------
	static function init() {

		// Pre-fill font definitions
		self::get_font_families();

		// Emit custom font css in head 
		add_action( 'wp_head', function(){ 
			echo self::get_font_css(); 
		},5);

		// Load CSS when we are in Gutenberg Editor. 
		// Requires ma_customfont.css (for font loading) and oxygen.css/universal.css (for font assignment)
		if ( isset($_REQUEST['post']) && isset($_REQUEST['action']) && ($_REQUEST['action']=='edit')) {
			// set up fonts dir and url
			$fonts_base = self::get_fonts_base(); 
			if (!$fonts_base) 	{return false;}
			// enqueue ma_customfonts.css
			wp_enqueue_style('ma_customfonts-gutenberg', $fonts_base->url.'/ma_customfonts.css'); 
			if (defined("CT_VERSION")) { // Oxygen installed and active?
				// create custom style for body, h1-h6 from Oxygen global settings
				$ct_global_settings = maybe_unserialize(get_option('ct_global_settings'));
				if ($ct_global_settings && is_array($ct_global_settings)) {
					$gutenberg_font_css = sprintf(
						'body .editor-styles-wrapper{font-family:"%1$s";}'.
						'body .editor-styles-wrapper :is(h1,h2,h3,h4,h5,h6) {font-family:"%2$s";}',
						$ct_global_settings['fonts']['Text'], 	// text font
						$ct_global_settings['fonts']['Display']	// heading font
					);
					// add custom style to overwrite Gutenberg's default font
					wp_add_inline_style('ma_customfonts-gutenberg',$gutenberg_font_css); 
				}
			}
		}
		
		
		// Shortcode for testing custom fonts (listing all fonts with their formats, weights, styles)
		add_shortcode('maltmann_custom_font_test', function( $atts, $content, $shortcode_tag ) {
			return self::get_font_samples('shortcode');
		}); 

		// Add submenu page to the Appearance menu.
		add_action('admin_menu', function(){
			add_submenu_page(	'themes.php', 										// parent slug of "Appearance"
								_x('Custom Fonts','page title','ma_customfonts'), 	// page title
								_x('Custom Fonts','menu title','ma_customfonts'), 	// menu title
								'manage_options',									// capabilitiy
								'ma_customfonts',									// menu slug
								[__CLASS__, 'admin_customfonts']							// function
							);
		});
	}

	//-------------------------------------------------------------------------------------------------------------------
	static function get_script_version() {
		$implementation = basename(__FILE__) == 'ma-custom-fonts.php' ? 'Plugin' : 'Code Snippet';
		return sprintf('%s, %s', $implementation, self::VERSION);
	}
	//-------------------------------------------------------------------------------------------------------------------
	// Admin function Appearance > Custom Fonts to display samples of all detected fonts
	static function admin_customfonts() {
		$output =	'<h1>' . esc_html(get_admin_page_title()) . '</h1>'.
					self::get_font_samples('admin');
		echo $output;
		echo self::get_font_css();
	}
	//-------------------------------------------------------------------------------------------------------------------
	// parses weight from a font file name (not used for Web Font Loader packages)
	static function parse_font_name($name) {
		// already in cache?
		if (array_key_exists($name,self::$fonts_details_cache)) {return self::$fonts_details_cache[$name];}
		
		$retval = (object)['name'=>$name, 'weight'=>400, 'style'=>'normal'];
		if (!self::$parsename) {return $retval;}
		$st = microtime(true);
		if (WP_DEBUG && self::$debug) {error_log(sprintf('%s::%s() parsing font file name: "%s"',__CLASS__,__FUNCTION__, $retval->name));}
		$weights = (object)[ // must match from more to less specific !!
			// more specific
			200 => '/[ \-]?(200|((extra|ultra)\-?light))/i',
			800 => '/[ \-]?(800|((extra|ultra)\-?bold))/i',
			600 => '/[ \-]?(600|([ds]emi(\-?bold)?))/i',
			// less specific
			100 => '/[ \-]?(100|thin)/i',
			300 => '/[ \-]?(300|light)/i',
			400 => '/[ \-]?(400|normal|regular|book)/i',
			500 => '/[ \-]?(500|medium)/i',
			700 => '/[ \-]?(700|bold)/i',
			900 => '/[ \-]?(900|black|heavy)/i',
			'var' => '/[ \-]?(VariableFont)/i',
		];
		$count = 0;
		// detect & cut style
		$new_name = preg_replace('/[ \-]?(italic|oblique)/i', '', $retval->name, -1, $count); 
		if ($new_name && $count) {
			$retval->name = $new_name;
			$retval->style = 'italic';
			if (WP_DEBUG && self::$debug) {error_log(sprintf('%s::%s() detected italic, new font family name: "%s"',__CLASS__,__FUNCTION__, $retval->name));}
		}
		// detect & cut weight
		foreach ($weights as $weight => $pattern) {
			$new_name = preg_replace($pattern, '', $retval->name, -1, $count);
			if ($new_name && $count) {
				$retval->name = $new_name;
				$retval->weight = $weight;
				if (WP_DEBUG && self::$debug) {error_log(sprintf('%s::%s() detected weight %s, new font family name: "%s"',__CLASS__,__FUNCTION__, $retval->weight, $retval->name));}
				break;
			}
		}
		// cut -webfont
		$retval->name = preg_replace('/[ \-]?webfont$/i', '', $retval->name); 
		// variable font: detect & cut specifica
		if ($retval->weight == 'var') {
			$retval->name = preg_replace('/_(opsz,wght|opsz|wght)$/i', '', $retval->name); 
		}
		if (WP_DEBUG && self::$debug) {error_log(sprintf('%s::%s() retval: [name:"%s", weigh:%d, style:%s]',__CLASS__,__FUNCTION__, $retval->name, $retval->weight, $retval->style));}
		// store to cache
		self::$fonts_details_cache[$name] = $retval;
		$et = microtime(true);
		if (WP_DEBUG && self::$timing) {error_log(sprintf('%s::%s() Timing: %.5f sec.',__CLASS__,__FUNCTION__,$et-$st));}
		return $retval;
	}
	//-------------------------------------------------------------------------------------------------------------------
	// construct CSS block from CSS properties stored in JSON from Web Font Loader
	static 	function create_css_from_ruleset($css_ruleset) {
		$retval = '';
		if (isset($css_ruleset)) {
			if (isset($css_ruleset->{'comment'})) {$retval .= sprintf("/* %s */\n",$css_ruleset->{'comment'});}
			$retval .= "@font-face {\n";
			$retval .= sprintf("\tfont-family: '%s';\n",$css_ruleset->{'font-family'});
			$retval .= sprintf("\tfont-style: %s;\n",$css_ruleset->{'font-style'});
			$retval .= sprintf("\tfont-weight: %s;\n",$css_ruleset->{'font-weight'});
			$retval .= sprintf("\tsrc: url('%s') format('%s');\n",$css_ruleset->{'url'}, $css_ruleset->{'format'});
			if (isset($css_ruleset->{'unicode-range'})) {$retval .= sprintf("\tunicode-range: %s;\n", $css_ruleset->{'unicode-range'});}
			if (self::$fontdisplay) {
				$retval .= sprintf("\tfont-display: %s;\n",self::$fontdisplay);
			}			
			$retval .= '}';
		}
		return $retval;
	}
	//-------------------------------------------------------------------------------------------------------------------
	// return base dir/url for fonts. Create directory if necessary
	private static function get_fonts_base() {
		$retval = (object)['dir'=>null,'url'=>''];
		$fonts_dir_info = wp_get_upload_dir();
		$retval->dir = $fonts_dir_info['basedir'].'/fonts';
		$retval->url = $fonts_dir_info['baseurl'].'/fonts';
		// create fonts folder if not exists
		if (!file_exists($retval->dir)) {
			if (!@mkdir($retval->dir)) {
				error_log(sprintf('%s::%s() Error creating fonts base folder.', __CLASS__, __FUNCTION__)); 
				return null;
			}
		}
		return $retval;
	}
	//-------------------------------------------------------------------------------------------------------------------
	// find font files in font folder
	static function find_fonts() {
		$st = microtime(true);
		if (isset(self::$fonts)) return;
		self::$fonts = [];
		// set up fonts dir and url
		$fonts_base = self::get_fonts_base(); 
		if (!$fonts_base) 	{return false;}
		// property $recursive either recursive or flat file scan
		if (self::$recursive) {
			// recursive scan for font files (including subdirectories)
			$directory_iterator = new RecursiveDirectoryIterator($fonts_base->dir,  RecursiveDirectoryIterator::SKIP_DOTS | RecursiveDirectoryIterator::UNIX_PATHS);
			$file_iterator = new RecursiveIteratorIterator($directory_iterator);
		} else {
			// flat scan for font files (no subdirectories)
			$file_iterator = new FilesystemIterator($fonts_base->dir);
		}
		// loop through files and collect font and JSON files
		$font_splfiles = [];
		$json_splfiles = [];
		foreach( $file_iterator as $file) {
			// V3: A JSON file might be available from Web Font Loader
			if ($file->getExtension() == 'json') {
				$json_splfiles[] = $file;
			}
			if (in_array(strtolower($file->getExtension()), self::$prioritized_formats)) {
				$font_splfiles[] = $file;
			}
		}
		
		// V3: check JSON files. If it defines "family" read the font name and CSS
		$json_font_families = [];
		foreach ($json_splfiles as $json_splfile) {
			if ($font_details = @json_decode(@file_get_contents($json_splfile->getPathname()))) {
				// It's a JSON from Web Font Loader?
				if (isset($font_details->creator) && (strpos($font_details->creator, 'Web Font Loader')=== 0)) {
					// store font family name 
					$json_font_families[$json_splfile->getBasename('.json')] = $font_details->family;
					// drop all collected font files for that font since they are listed in JSON file
					$font_path = $json_splfile->getPath().'/';
					foreach ($font_splfiles as $idx => $font_splfile) {
						if (strpos($font_splfile->getPath().'/',$font_path) === 0) {
							self::$font_files_cnt ++;
							unset($font_splfiles[$idx]);
						}
					}
					$font_path = str_replace($fonts_base->dir,'',$font_path);
					// encode every single path element since we might have spaces or special chars 
					$font_path = implode('/',array_map('rawurlencode',explode('/',$font_path)));
					
					// add CSS blocks (could be multiple unicode ranges) to fonts list

					$font_baseurl = $fonts_base->url . $font_path;
					foreach ($font_details->css as $css_ruleset) {
						self::$fonts[$css_ruleset->{'font-family'}][$css_ruleset->{'font-weight'}.'/'.$css_ruleset->{'font-style'}]['has_css'] = true;
						// only formats woff and woff2, so just use format as file extension slot
						if (!isset(self::$fonts[$css_ruleset->{'font-family'}][$css_ruleset->{'font-weight'}.'/'.$css_ruleset->{'font-style'}][$css_ruleset->{'format'}])) {
							self::$fonts[$css_ruleset->{'font-family'}][$css_ruleset->{'font-weight'}.'/'.$css_ruleset->{'font-style'}][$css_ruleset->{'format'}] = [];	
						}
						$css_ruleset->url = $font_baseurl . $css_ruleset->url;

						$css_block = self::create_css_from_ruleset($css_ruleset);
						self::$fonts[$css_ruleset->{'font-family'}][$css_ruleset->{'font-weight'}.'/'.$css_ruleset->{'font-style'}][$css_ruleset->{'format'}][] = $css_block;
					}
				}
			}
		}
		// collect font definitions
		foreach ($font_splfiles as $font_splfile) {
			self::$font_files_cnt ++;
			$font_ext = $font_splfile->getExtension();
			$font_details = self::parse_font_name($font_splfile->getbasename('.'.$font_ext));
			$font_name = $font_details->name;
			if (in_array($font_name,array_values($json_font_families))) {
				// already found this font from Web Font Loader. Skip.
				continue;
			}
			$font_weight = $font_details->weight;
			$font_style = $font_details->style;
			$font_path = str_replace($fonts_base->dir,'',$font_splfile->getPath());
			// encode every single path element since we might have spaces or special chars 
			$font_path = implode('/',array_map('rawurlencode',explode('/',$font_path)));
			// create entry for this font name
			if (!array_key_exists($font_name,self::$fonts)) {self::$fonts[$font_name] = [];}
			// create entry for this font weight/style 
			if (!array_key_exists($font_weight.'/'.$font_style,self::$fonts[$font_name])) {self::$fonts[$font_name][$font_weight.'/'.$font_style] = [];}
			// store font details for this file
			self::$fonts[$font_name][$font_weight.'/'.$font_style][$font_ext] = $fonts_base->url . $font_path . '/' . rawurlencode($font_splfile->getBasename());
		}
		ksort(self::$fonts, SORT_NATURAL | SORT_FLAG_CASE);
		if (WP_DEBUG && self::$debug) {error_log(sprintf('%s::%s() final fonts: %s]',__CLASS__,__FUNCTION__, print_r(self::$fonts,true)));}
		$et = microtime(true);
		if (WP_DEBUG && self::$timing) {error_log(sprintf('%s::%s() %d font files, %d font families.',__CLASS__,__FUNCTION__, self::$font_files_cnt, count(self::$fonts)));}
		if (WP_DEBUG && self::$timing) {error_log(sprintf('%s::%s() Timing: %.5f sec.',__CLASS__,__FUNCTION__,$et-$st));}
	}
	//-------------------------------------------------------------------------------------------------------------------
	// returns a list of font families
	static function get_font_families() {
		if (!isset(self::$fonts)) self::find_fonts();
		$st = microtime(true);
		$font_family_list = [];
		foreach (array_keys(self::$fonts) as $font_name) {
			$font_family_list[] = $font_name;
		}
		$et = microtime(true);
		if (WP_DEBUG && self::$timing) {error_log(sprintf('%s::%s() Timing: %.5f sec.',__CLASS__,__FUNCTION__,$et-$st));}
		return $font_family_list;
	}
	//-------------------------------------------------------------------------------------------------------------------
	// we call this function from footer emitter to get font definitions for emitting required files
	static function get_font_definitions() {
		return self::$fonts;
	}
	//-------------------------------------------------------------------------------------------------------------------
	// creates and emits CSS for custom fonts
	static function get_font_css() {
		// emit CSS for fonts in footer
		$st = microtime(true);
		$version = self::get_script_version();
		$style = '';
		// set up fonts dir and url
		$fonts_base = self::get_fonts_base(); 
		if (!$fonts_base) 	{return false;}
		foreach (self::$fonts as $font_name => $font_details) {
			ksort($font_details);
			foreach ($font_details as $weight_style => $file_list) {
				list ($font_weight,$font_style) = explode('/',$weight_style);

				if (isset($file_list['has_css'])) {
					// V3: Google Font package CSS from Web Font Loader already has CSS
					foreach (array_reverse(self::$prioritized_formats) as $font_ext) {
						// we only have woff and woff2
						if (!isset($file_list[$font_ext])) { continue; }
						foreach ($file_list[$font_ext] as $css) {
							$style .= trim($css).PHP_EOL;
						}
					}
				} else {
					// V2: Only have font info and file names. Build CSS
					if ($font_weight == 'var') {
						$font_weight_output = '100 900';
					} else {
						$font_weight_output = $font_weight;
					}
					$style .= 	'@font-face{'.PHP_EOL.
								'  font-family:"'.$font_name.'";'.PHP_EOL.
								'  font-weight:'.$font_weight_output.';'.PHP_EOL.
								'  font-style:'.$font_style.';'.PHP_EOL;
								// .eot needs special handling for IE9 Compat Mode
					if (array_key_exists('eot',$file_list)) {$style .= '  src:url("'.$file_list['eot'].'");'.PHP_EOL;}
					$urls = [];

					// output font sources in prioritized order
					foreach (self::$prioritized_formats as $font_ext) {
						if (array_key_exists($font_ext,$file_list)) {
							$font_url = $file_list[$font_ext];
							$format = '';
							switch ($font_ext) {
								case 'eot': $format = 'embedded-opentype'; break;
								case 'otf': $format = 'opentype'; break;
								case 'ttf': $format = 'truetype'; break;
								// others have same format as extension (svg, woff, woff2)
								default:	$format = strtolower($font_ext);
							}
							if ($font_ext == 'eot') {
								// IE6-IE8
								$urls[] = 'url("'.$font_url.'?#iefix") format("'.$format.'")';
							} else {
								$urls[] = 'url("'.$font_url.'") format("'.$format.'")';
							}
						}
					}
					$style .= '  src:' . join(','.PHP_EOL.'      ',$urls) . ';'.PHP_EOL;
					if (self::$fontdisplay) {
						$style .= sprintf('  font-display: %s;'.PHP_EOL,self::$fontdisplay);
					}
					$style .= '}'.PHP_EOL;
				}
			}
		}
		// if Oxygen Builder is active, emit CSS to show custom fonts in light blue.
		$builder_style = defined('SHOW_CT_BUILDER') ? 'div.oxygen-select-box-option.ng-binding.ng-scope[ng-repeat*="elegantCustomFonts"] {color:lightblue !important;}' : '';
		
		
		if (WP_DEBUG && self::$debug) {error_log(sprintf('%s::%s() style: %s]',__CLASS__,__FUNCTION__, $style));}
		// minimize string if configured
		if (self::$cssminimize) {
			$style = preg_replace('/\r?\n */','',$style); 
		}

		$retval = '';
		if (self::$cssoutput == 'file') {
			// option: write CSS to file
			$css_path = $fonts_base->dir.'/ma_customfonts.css';
			file_put_contents($css_path, '/* Version: '.$version.' */'.PHP_EOL.$style);
			$css_url = str_replace($fonts_base->dir,$fonts_base->url ,$css_path);
			$retval = sprintf('<link id="MA_CustomFonts" itemprop="stylesheet" href="%s?%s" rel="stylesheet" type="text/css" />%s',$css_url, hash_file('CRC32', $css_path, false), $builder_style?'<style>'.$builder_style.'</style>':'');
		}
		if (self::$cssoutput == 'html') {
			// option: write CSS to html
			$retval = '<style id="MA_CustomFonts">'.'/* Version: '.$version.' */'.PHP_EOL.$style.PHP_EOL.$builder_style.'</style>';
		}


		$et = microtime(true);
		if (WP_DEBUG && self::$timing) {error_log(sprintf('%s::%s() Timing: %.5f sec.',__CLASS__,__FUNCTION__,$et-$st));}
		return $retval;
	}
	//-------------------------------------------------------------------------------------------------------------------
	static function get_font_file_info_from_css($css) {
		$retval = [];
		if (!is_array($css)) {$css = [$css];}
		foreach($css as $css_block) {
			if (preg_match('/url\(\'(.*?)\'\)/',$css_block,$matches)) {
				$retval[] = $matches[1];
			}
		}
		$retval = array_unique($retval); 
		return $retval;
	}
	//-------------------------------------------------------------------------------------------------------------------
	// returns HTML code to display all registered custom fonts
	// $mode 'admin':		formatting to be displayed on WP Admin > Appearance
	// $mode 'shortcode':	formatting to be displayed as shortcode output
	static function get_font_samples($mode = null) {
		$st = microtime(true);
		$output = '';
		$script_version = self::get_script_version();
		$output = 	'<style>'.
					'#ma_customfonts-input-font-size {width:60px;text-align:center;min-height:1em;line-height:1em;padding:0;}'.
					'#ma_customfonts-input-sample-text {width:400px;text-align:left;}'.
					'.ma_customfonts-label {display:inline-block;width:150px;line-height:2em;}'.
					'.ma_customfonts-font-row {display:flex;flex-direction:row;justify-content:space-between;align-items:center;padding:0;line-height:20px;border-bottom:1px solid #e0e0e0;margin:0 1em;}'.
					'.ma_customfonts-font-row:hover {background-color:lightgray;}'.
					'.ma_customfonts-font-info {font-size:10px;line-height:1em;width:100px;}'.
					'.ma_customfonts-font-sample {font-size:15px;line-height:1em;flex-grow:1;-webkit-font-smoothing:antialiased;-moz-osx-font-smoothing:grayscale;}'.
					'.ma_customfonts-format-info {font-size:10px;cursor:help;margin-left:1em;}'.
					'.ma_customfonts-simulated {display: none;}'.
					'</style>'.
					'<div '.($mode=='shortcode'?'style="display:inline-block;border:1px dashed darkgray;padding:10px;"':'').'>'.
						($mode=='shortcode'?'<h2>MA Custom Fonts</h2>':'').
						'<div style="display:inline-block;border:1px solid darkgray;border-radius:10px;padding:10px;">'.
							'<span class="ma_customfonts-label">Version:</span> '.$script_version.'<br/>'.
							'<span class="ma_customfonts-label">Font Families:</span> '.count(self::$fonts).'<br/>'.
							'<span class="ma_customfonts-label">Font Files:</span> '.self::$font_files_cnt.'<br/>'.
							'<span class="ma_customfonts-label">Sample Font Size:</span> '.
								'<input id="ma_customfonts-input-font-size" type="number" value="15" onchange="ma_customfonts_change_font_size();"><br/>'.
							'<span class="ma_customfonts-label">Sample Text:</span> '.
								'<input id="ma_customfonts-input-sample-text" value="'.self::$sample_text.'" onkeyup="ma_customfonts_change_sample_text();"><br/>'.
							'<span class="ma_customfonts-label">Simulated:</span> '.
								'<input id="ma_customfonts-input-simulated" type="checkbox" value="simulated" onchange="ma_customfonts_toggle_simulated();"> Show font weights/styles without files as browser would simulate.<br/>'.
						'</div>';

		// controls
		$controls_script = <<<'END_OF_CONTROLS_SCRIPT'
		<script>
		function changeCss(className, classValue) {
			// we need invisible container to store additional css definitions
			var cssMainContainer = jQuery('#css-modifier-container');
			if (cssMainContainer.length == 0) {
				cssMainContainer = jQuery('<div id="css-modifier-container"></div>');
				cssMainContainer.hide().appendTo(jQuery('body'));
			}
			// we need one div for each class
			var classContainer = cssMainContainer.find('div[data-class="' + className + '"]');
			if (classContainer.length == 0) {
				classContainer = jQuery('<div data-class="' + className + '"></div>');
				classContainer.appendTo(cssMainContainer);
			}
			// append additional style
			classContainer.html('<style>' + className + ' {' + classValue + '}</style>');
		}
		function ma_customfonts_change_font_size() {
			var $val = jQuery('#ma_customfonts-input-font-size').val();
			changeCss('.ma_customfonts-font-sample','font-size: '+$val+'px;');
		}
		function ma_customfonts_change_sample_text() {
			var $val = jQuery('#ma_customfonts-input-sample-text').val();
			jQuery('.ma_customfonts-font-sample').text($val);
		}
		function ma_customfonts_toggle_simulated() {
			var $simulated = jQuery('#ma_customfonts-input-simulated').is(':checked');
			jQuery('.ma_customfonts-simulated').css('display',$simulated?'flex':'none');
		}
		</script>
END_OF_CONTROLS_SCRIPT;
		$output .=  $controls_script;
		// prepare tags for every weight/style combination
		$weights = [100,200,300,400,500,600,700,800,900];
		$styles = ['normal','italic'];
		$weights_styles = [];
		foreach ($weights as $weight) { foreach ($styles as $style) { $weights_styles[] = $weight.'/'.$style; } }
		// display fonts in each weight/style combination
		$sample_text = self::$sample_text;

		// build output
		foreach (self::$fonts as $font_name => $font_details) {
			$output .= sprintf('<h3 style="padding-top: 20px;">%1$s</h3>',$font_name);
			ksort($font_details);
			foreach ($weights_styles as $weight_style) {
				list ($weight,$style) = explode('/',$weight_style);;
				$font_file_info = '';
				$font_file_list = [];
				if (isset($font_details[$weight_style])) {
					// walk through possible file formats
					foreach (self::$prioritized_formats as $font_ext) {
						// details available for this specific file format?
						if (isset($font_details[$weight_style][$font_ext])) {
							// details content type
							if (isset($font_details[$weight_style]['has_css'])) {	
								// CSS
								$font_file_list[$font_ext] = self::get_font_file_info_from_css($font_details[$weight_style][$font_ext]);
							} else {
								// just file name
								$font_file_list[$font_ext] = [$font_details[$weight_style][$font_ext]];
							}
						}
					}
					// build font file info output
					foreach ($font_file_list as $format => $files) {
						// cut leading path/url from file info
						$files = str_replace(wp_get_upload_dir(),'',$files);
						// decode html entities (e.g. %20) in file path
						foreach ($files as &$file) {$file = implode('/',array_map('rawurldecode',explode('/',$file)));;}
						// convert array to html
						$font_file_list[$format] = sprintf('<span title="%2$s">%1$s</span>', strtoupper($format), implode("\n",$files));
					}
					$font_file_info = '<span class="ma_customfonts-format-info">(' . implode(', ',array_values($font_file_list)) . ')</span>';
				}

				$output .= sprintf(	'<div class="ma_customfonts-font-row '.($font_file_info?'':'ma_customfonts-simulated').'">'.
										'<span class="ma_customfonts-font-info">%2$s %3$s</span>'.
										'<span class="ma_customfonts-font-sample" style="font-family:\'%1$s\';font-weight:%2$d;font-style:%3$s">%4$s</span>%5$s'.
									'</div>',$font_name, $weight, $style, $sample_text, $font_file_info?$font_file_info:'<span class="ma_customfonts-format-info"><em>(simulated)</em></span>');

			}
		}
		$output .= '</div>';

		$et = microtime(true);
		if (WP_DEBUG && self::$timing) {error_log(sprintf('%s::%s() Timing: %.5f sec.',__CLASS__,__FUNCTION__,$et-$st));}
		return $output;

	}
} // end of class MA_CustomFonts


endif; // end of conditional implementations


// check if we have to run
$run = true;
$ajax = wp_doing_ajax();
$cron = wp_doing_cron();
if ($ajax) $run = false;		// don't run for AJAX requests
if ($cron) $run = false;		// don't run for CRON requests
#if (preg_match('/\.(ico|gif|png|jpg|jpeg)$/i',@$_SERVER['REQUEST_URI'])) $run = false; // don't run for WP generated images
if (preg_match('/\.(ico)$/i',@$_SERVER['REQUEST_URI'])) $run = false; // don't run for WP generated images
// output the request for optimization process
if (WP_DEBUG && (MA_CustomFonts::$debug || MA_CustomFonts::$timing)) {error_log(sprintf('MA_CustomFonts: %s%sRequest action="%s", URI="%s" => run: %s', $ajax?'AJAX ':'', $cron?'CRON ':'',@$_REQUEST['action'], @$_SERVER['REQUEST_URI'], $run?'true':'false'));}
if (!$run) return;


// Initialize
add_action('plugins_loaded',function(){

	MA_CustomFonts::init();
});

//-------------------------------------------------------------------------------------------------------------------
// Warn if plugins "Elegant Custom Forms", "Use Any Font", "Swiss Knife / Font Manager" are active
add_action('wp_loaded',function(){
	$GLOBALS['ma_custom_fonts_incompatible_plugins'] = [];
	if (function_exists('is_plugin_active') && is_plugin_active('elegant-custom-fonts/elegant-custom-fonts.php'))
		{$GLOBALS['ma_custom_fonts_incompatible_plugins'][] = '"Elegant Custom Fonts"';}
	if (function_exists('is_plugin_active') && is_plugin_active('use-any-font/use-any-font.php'))
		{$GLOBALS['ma_custom_fonts_incompatible_plugins'][] = '"Use Any Font"';}
	if (function_exists('is_plugin_active') && is_plugin_active('swiss-knife/swiss-knife.php') && (get_option('swiss_font_manager')=='yes')) 														
		{$GLOBALS['ma_custom_fonts_incompatible_plugins'][] = '"Swiss Knife" with feature "Font Manager" enabled';}
	// show message on incompatible plugins
	if (is_admin()) {
		if (count($GLOBALS['ma_custom_fonts_incompatible_plugins'])) {
			add_action('admin_notices', function(){
				if (WP_DEBUG ) {error_log('$ma_custom_fonts_incompatible_plugins: '.print_r($GLOBALS['ma_custom_fonts_incompatible_plugins'],true));}
				echo '<div class="notice notice-warning is-dismissible">
						<p>The Code Snippet "Oxygen: Custom Fonts" is not compatible with the Plugin '.implode(' or ',$GLOBALS['ma_custom_fonts_incompatible_plugins']).'.<br/>
						Please deactivate either the Code Snippet or the Plugin (feature).</p>
					</div>';
			});
		}
	}

	if (count($GLOBALS['ma_custom_fonts_incompatible_plugins'])) return;
	

	//-------------------------------------------------------------------------------------------------------------------
	// create a primitive ECF_Plugin class if plugin "Elegant Custom Fonts" is not installed
	if (!count($GLOBALS['ma_custom_fonts_incompatible_plugins']) && !class_exists('ECF_Plugin')) {
		class ECF_Plugin {
			static function get_font_families() {
				$st = microtime(true);
				$font_family_list = MA_CustomFonts::get_font_families();
				$et = microtime(true);
				if (WP_DEBUG && MA_CustomFonts::$timing) {error_log(sprintf('MA_CustomFonts/%s::%s() Timing: %.5f sec.',__CLASS__,__FUNCTION__,$et-$st));}
				return $font_family_list;
			}
		}
	}
	

},1000); // hook late to check other plugins!
