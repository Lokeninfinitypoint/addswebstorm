![alt text](image.png)# AI Coding Agent Instructions for this Repository

These instructions summarize project-specific patterns so an AI agent can be productive quickly. Focus work inside plugins and theme code; avoid editing WordPress core files (`wp-includes/`, `wp-admin/`).

## Architecture & Key Domains
- WordPress site with: custom Iteck theme (`wp-content/themes/iteck`), its companion plugin (`wp-content/plugins/iteck_plugin`), a custom Adalysis theme (`wp-content/themes/adalysis`), WooCommerce, numerous optimization / AI plugins, and generated static cache HTML files (`index.html?p=ID.html`). Treat cached HTML as build artifacts; never hand‑edit them—change source templates or theme/plugin PHP.
- Iteck theme options & UI customization use Redux Framework. Option definition files live under `wp-content/themes/iteck/inc/options/*.php` (e.g. `general.php`, `header.php`, `footer.php`, `woocommerce.php`). Each file calls `Redux::setSection($iteck_pre, ...)` composing the global options array referenced via the option name `iteck_theme_setting` (see screenshot opt_name). Extend options by adding a new `Redux::setSection` call in an appropriate file or a new file under this directory.
  - Conditional rendering (headers, side panels, footers) is routed through actions in `iteck-builder.php` (template for page/portfolio types). It checks `class_exists('ReduxFrameworkPlugin')` before invoking `do_action('iteck-custom-header', ...)` etc. Hook new dynamic UI into those custom action names rather than editing template logic.
  - Companion plugin (`iteck_plugin/iteck_plugin.php`) loads Elementor addons, CPTs (`portfolio`), meta boxes, custom header/footer/side panel implementations, widgets, importer, and license logic. Add enhancements by appending `include()` statements or hooking after `plugins_loaded` inside this plugin instead of the theme when you need plugin scope.
- WooCommerce: Custom theme options for WooCommerce cart icon and other presentation live in `inc/options/woocommerce.php` (e.g. `iteck_header_cart`). Template overrides should be placed in the theme under `woocommerce/` (add directory if missing). Cart/Checkout/Shop pages in admin are dynamic; avoid editing cached HTML duplicates.
- Core AI/MCP integration lives in `wp-content/plugins/wordpress-mcp/`.
  - Entry point: `wordpress-mcp.php` (strict types, singleton access via `WPMCP()` -> `WpMcp::instance()` on `plugins_loaded`).
  - Orchestrator: `includes/Core/WpMcp.php` registers Tools, Resources, Prompts; defers REST init until `rest_api_init` (priority 20000) when MCP is enabled in `wordpress_mcp_settings` option.
  - Transports: `McpStdioTransport` & `McpStreamableTransport` expose MCP channels (STDIO / streamable) when plugin initializes.
  - Feature flags: `enable_create_tools`, `enable_update_tools`, `enable_delete_tools`, `enable_rest_api_crud_tools`, `features_adapter_enabled`. Always branch on these before expensive work or registrations.
- Hostinger AI plugin (`hostinger-ai-assistant`) registers its own MCP endpoints/classes after `plugins_loaded`. Avoid naming collisions in REST routes and duplicate AI tool semantics.
- Custom data modeling / rendering: `secure-custom-fields` plugin (`SCF\...`) enhances ACF Flexible Content via `Render.php` and `Layout.php`. Use those classes for flexible content rather than duplicating markup.

## Plugin Development Conventions
- Namespacing: Use PHP namespaces matching directory structure (e.g. `Automattic\WordpressMcp\Tools\MyTool`). Avoid global functions except intentional singletons or WordPress hook callbacks.
- Initialization: Register new functionality inside constructors called during `WpMcp` default init sequence or via the `wordpress_mcp_init` action (fires once after REST is ready and MCP enabled). Prefer hooking late if you depend on other plugins' REST routes.
- Tool/Resource/Prompt registration:
  - Tools: `WpMcp::register_tool( [ 'name' => ..., 'type' => 'read|create|update|delete|action', 'description' => ..., 'input_schema' => ..., 'callback' => callable, 'permission_callback' => callable, 'rest_alias' => optional ] )`.
  - Resources: `register_resource( [ 'name' => ..., 'uri' => unique, 'description' => ..., 'schema' => ... ] )` followed by `register_resource_callback( $uri, $callable )`. URIs are normalized to lowercase; ensure uniqueness.
  - Prompts: `register_prompt( $prompt_array, $messages_array )` where `$prompt_array['name']` is unique.
  - When REST CRUD tools are enabled (`enable_rest_api_crud_tools`), tools with `rest_alias` or flagged `disabled_by_rest_crud` are skipped—do not rely on their callbacks executing.
- Enabled/disabled state: Tool visibility is controlled by stored option `wordpress_mcp_tool_states`; use `is_tool_enabled( $name )` before performing expensive work.
- Strict types: Maintain `declare(strict_types=1);` in new PHP source for MCP plugin consistency.

## Frontend / Build Workflows
- `wordpress-mcp` JS admin/settings UI uses `@wordpress/scripts` (see `package.json`). Typical commands:
  - Install deps: `npm install` (or `pnpm install` if a lockfile is added later).
  - Build production assets: `npm run build`.
  - Dev watch: `npm start`.
  - Distribution zip: `composer install --no-dev && npm run build && npm run plugin-zip`.
- `secure-custom-fields` relies on Composer autoload + dev tooling:
  - Install: `composer install`.
  - Lint: `composer lint` (runs PHPCS with WPCS rules).
  - Format: `composer format` (PHPCBF + package normalization).
  - Tests: `composer test` (PHPUnit + phpstan analyze). Baseline generation: `composer test:phpstan:baseline`.
- Ensure vendor autoload exists before plugin init; the MCP bootstrap halts with `wp_die` if `vendor/autoload.php` missing.

## Theme & Static Pages
- Cached HTML `index.html?p=ID.html` pages (WP Rocket) are output snapshots. Never treat them as canonical; adjust theme templates, Elementor blocks, or Redux options instead.
- Iteck theme option workflow: values saved under Redux opt name propagate into front‑end via hooked actions. For new front‑end behaviors, prefer: add Redux field -> consume via existing action (`iteck-custom-header`, `iteck-custom-footer`, `iteck-custom-sidepanel`) instead of hardcoding in templates.
- Adding theme options: create a new section in `inc/options/*.php`; use unique `id` and simple field `type` (select, button_set, color, typography). Follow existing examples for conditional `required` arrays.
- Child‑safe modifications: whenever overriding header/footer markup, hook into provided actions instead of editing core template `iteck-builder.php`. If you must alter layout container, keep the existing `while (have_posts()) : the_post();` loop intact.
- Fonts & assets: Both `adalysis` and `iteck` themes reference fonts & images under their respective `assets/` directories. Maintain paths to avoid cache invalidation; prefer adding new assets rather than renaming existing ones.
- WooCommerce template overrides: place files under `wp-content/themes/iteck/woocommerce/`. Avoid direct edits inside the `woocommerce` plugin directory.
- Elementor & Iteck plugin: custom widgets/components live inside `iteck_plugin/inc/`. Extend Elementor features by adding new widget classes there and including them through `inc/elementor-addon.php` rather than modifying vendor code.

## Adding New MCP Functionality (Example)
1. Create a class `includes/Tools/McpMyCustomTool.php` under namespace `Automattic\WordpressMcp\Tools`.
2. In its constructor call `WpMcp::instance()->register_tool( ... )` with proper `type` and schema. Provide a small `permission_callback`—typically capability check (`current_user_can( 'manage_options' )`).
3. Hook any REST dependency after `wordpress_mcp_init` to ensure namespace `$wp_mcp->get_namespace()` is available.
4. If exposing existing REST endpoints as tools, set `rest_alias` so CRUD enabling logic manages conflicts.

## Safe Modification Practices
- Never edit WordPress core (`wp-admin/`, `wp-includes/`) or directly modify plugin vendor folders.
- Respect MCP feature flags and tool enabled states before registration/execution.
- For Iteck theme: avoid editing generated Redux option arrays programmatically after they load; add or remove via static `Redux::setSection` calls.
- Use SCF Flexible Content helpers (`SCF\Fields\FlexibleContent\Render`, `Layout`) for ACF Flexible fields.
- Keep license keys, secrets (e.g., theme purchase code) out of version control—do not hardcode them in source.

## Debugging & Inspection
- Check MCP state: `get_option('wordpress_mcp_settings')['enabled']`.
- Inspect tools/resources: `WpMcp::instance()->get_tools()`, `get_all_tools()` (includes `enabled`), `get_resources()`.
- Redux option inspection: dump `get_option('iteck_theme_setting')` to view saved theme settings; match keys with `id` values in `inc/options/*.php` (e.g. `iteck_header_cart`, `iteck_main_color`).
- WooCommerce presentation issues: verify header cart icon controlled by `iteck_header_cart` setting; ensure custom templates override logic not broken by caching.

## Deployment / Packaging Notes
- Production baselines: PHP ≥ 8.0 (plugins require), WordPress ≥ 6.4 (MCP plugin header). Some legacy plugins may tolerate 6.5.x.
- MCP plugin distribution: `composer install --no-dev && npm run build && npm run plugin-zip` (or `pnpm` if lockfile added). Ensure vendor autoload present or bootstrap will `wp_die`.
Theme/plugin licensing: keep purchase/license codes in `.private/` or environment, not committed.

## Remote Access & WP-CLI Diagnostics
SSH (non‑standard port supplied): `ssh -p 65002 u520004865@77.37.90.129` (replace user/host as needed). Run WP-CLI from WordPress root (`wp-config.php` directory).

Baseline commands:
```
wp core verify-checksums
wp core version
wp plugin status
wp theme status
wp option get siteurl && wp option get home
wp config list
wp rewrite list
wp cron event list
wp db check && wp db optimize
wp transient list --fields=key,expiry
wp cache flush
wp plugin update --all --dry-run
wp theme update --all --dry-run
```

Debug & logs:
- Keep `WP_DEBUG` & `WP_DEBUG_LOG` disabled in production; enable temporarily for targeted issues.
- Inspect `wp-content/debug.log` (tail last lines) and hosting `error_log` files.
- Large autoload options: `wp option list --autoload=on --fields=option_name,size | sort -nr -k2 | head -n 20`.

Security quick wins:
- Verify no unexpected admin users: `wp user list --role=administrator --fields=ID,user_login,user_email`.
- If checksum mismatches: `wp core download --force` after backup.

Performance snapshot:
- Cron health: ensure no overdue events; run due: `wp cron event run --due-now`.
- Transient bloat: review long‑expired items; avoid mass deletion if caching layer depends on them.

## Operational Maintenance & Audit Checklist
Monthly or pre‑deployment:
1. Menus & navigation: load primary pages; ensure no 404 network calls or console JS errors.
2. Broken links: temporary link checker plugin or external crawl; remove tool afterward.
3. Cache coherence: purge WP Rocket/Hostinger caches after structural/theme option changes.
4. WooCommerce pages: confirm cart/checkout/account pages mapped; header cart icon -> Redux `iteck_header_cart`.
5. MCP flags: audit `wordpress_mcp_settings` for unused transports.
6. Security hygiene: remove unused themes/plugins; audit admin list.
7. DB/autoload: refactor any >100KB autoload options to lazy load.
8. Assets: ensure no font/image 404s; version new asset files with query strings.
9. Backups: test restore to staging quarterly.
10. Cron reliability: no stuck tasks; reschedule if needed.

## Sample One-Off Diagnostic Script (Delete After Use)
Create `diag-report.php` in site root (temporary):
```php
<?php declare(strict_types=1); require __DIR__ . '/wp-load.php'; header('Content-Type: text/plain'); if(!current_user_can('manage_options')){http_response_code(403);exit('Forbidden');}
global $wpdb; echo "== SITE ==\n"; echo 'Home: '.home_url()."\n"; echo 'Version: '.get_bloginfo('version')."\n"; $theme=wp_get_theme(); echo "Theme: {$theme->get('Name')} {$theme->get('Version')}\n"; echo "\n== PLUGINS ==\n"; foreach(get_option('active_plugins',[]) as $p){$d=get_plugin_data(WP_PLUGIN_DIR.'/'.$p,false,false); echo $d['Name'].' '.$d['Version']."\n";} echo "\n== CRON (15) ==\n"; $crons=_get_cron_array(); $c=0; foreach($crons as $ts=>$hooks){ foreach($hooks as $h=>$evs){ foreach($evs as $sig=>$ev){ printf("%s | %s | %s\n", date('Y-m-d H:i:s',(int)$ts), $h, $ev['schedule']??'once'); if(++$c>=15) break 3; } } } echo "\n== LARGE AUTOLOAD (>100KB) ==\n"; $rows=$wpdb->get_results("SELECT option_name,LENGTH(option_value) size FROM {$wpdb->options} WHERE autoload='yes' ORDER BY size DESC LIMIT 20"); foreach($rows as $r){ if($r->size>102400) echo $r->option_name.' '.round($r->size/1024,1).'KB' . "\n"; } echo "\n== REDUX SNAPSHOT ==\n"; $redux=get_option('iteck_theme_setting'); if(is_array($redux)) foreach($redux as $k=>$v){ if(is_scalar($v)) echo "$k=$v\n"; } echo "\n== DONE ==\n"; /* Remove file after use */
```
Security: restrict access (optional random filename, delete immediately). Never leave diagnostics scripts live.


- Do not edit `woocommerce` core templates—override in theme.

Feedback welcome: Clarify any missing workflow, schema field, or cross-plugin interaction you need before deeper changes.
