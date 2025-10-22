# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Pressable Cache Management is a WordPress plugin designed specifically for the Pressable hosting platform. It provides an admin interface to manage Batcache (object cache), Edge Cache, and various cache-related settings without requiring access to the Pressable control panel.

**Important**: This plugin only functions on sites hosted on Pressable (`IS_PRESSABLE` constant must be defined). It will auto-deactivate on other platforms.

## Architecture

### Plugin Entry Point
- **Main file**: [pressable-cache-management.php](pressable-cache-management.php) - Loads all modules and defines plugin metadata
- **Version**: 5.2.2 (current)
- **Text domain**: `pressable_cache_management`

### Core Module System

The plugin uses a modular architecture with clear separation between admin UI and functionality:

#### Admin Interface (`admin/`)
- `admin-menu.php` - Creates top-level admin menu (displays as "Pressable CM" or "Cache Control" based on branding settings)
- `settings-page.php` - Renders the tabbed settings interface
- `settings-register.php` - Registers all settings fields and sections using WordPress Settings API
- `settings-callbacks.php` - Callback functions for rendering form fields
- `settings-validate.php` - Input validation and sanitization

#### Custom Functions (`admin/custom-functions/`)

**Cache Control Functions:**
- `flush_object_cache.php` - Flushes WordPress object cache (database cache)
- `purge_edge_cache.php` - Purges Edge Cache via Edge_Cache_Plugin class
- `turn_on_off_edge_cache.php` - Enable/disable Edge Cache via Edge_Cache_Plugin class
- `object_cache_admin_bar.php` - Adds cache flush button to WordPress admin bar
- `flush_single_page_toolbar.php` - Adds per-page cache flush to admin toolbar

**Batcache Extensions:**
- `extend_batcache.php` - Extends Batcache storage from default to 24 hours by creating mu-plugin
- `exclude_pages_from_batcache.php` - Excludes specific pages from Batcache and Edge Cache

**Automatic Cache Flushing:**
- `flush_cache_on_page_edit.php` - Auto-flush when pages/posts/custom post types are updated
- `flush_cache_on_page_post_delete.php` - Auto-flush when published content is deleted
- `flush_cache_on_comment_delete.php` - Auto-flush when comments are deleted
- `flush_cache_on_theme_plugin_update.php` - Auto-flush on theme/plugin updates

**WooCommerce Integration:**
- `flush_batcache_for_woo_individual_page.php` - Flushes cache for individual WooCommerce product pages
- `flush_batcache_for_particular_page.php` - Flushes Batcache for specific pages

**Utilities:**
- `wp-write-to-file-lib.php` - Library for safely modifying wp-config.php (used for legacy operations)
- `remove_pressable_branding.php` - Controls plugin branding visibility

### MU-Plugin System

The plugin dynamically creates must-use (mu-plugins) for certain features to ensure they load before all other plugins:

1. Creates `/mu-plugins/pressable-cache-management.php` as index file
2. Creates `/mu-plugins/pressable-cache-management/` directory for feature-specific mu-plugins
3. Copies template files (`*_mu_plugin.php`) from plugin directory to mu-plugins when features are enabled
4. Automatically removes mu-plugins when features are disabled

**MU-Plugin Templates:**
- `extend_batcache_mu_plugin.php` - Extends Batcache max_age to 86400 seconds (24 hours)
- `exclude_pages_from_batcache_mu_plugin.php` - Implements page exclusion logic
- `pressable_cache_management_mu_plugin_index.php` - Index loader for mu-plugins directory

### Edge Cache Integration

The plugin integrates with Pressable's Edge Cache system via the `Edge_Cache_Plugin` class (provided by Pressable hosting):

- Checks if `Edge_Cache_Plugin` class exists before operations
- Uses `query_ec_backend('on'/'off')` for enable/disable operations
- Uses `purge_domain_now()` for cache purging
- Uses `get_ec_status()` to check current Edge Cache state
- Stores Edge Cache status in options: `edge-cache-enabled` and `edge-cache-status`

### Settings Tabs

The plugin UI is organized into three tabs:

1. **Object Cache Management** (`pressable_cache_management`) - Primary tab with Batcache and object cache settings
2. **Edge Cache Settings** (`edge_cache_settings_tab`) - Edge Cache enable/disable and purge controls
3. **Branding** (`remove_pressable_branding_tab`) - Show/hide Pressable branding in admin

### Cleanup and Maintenance

- `remove_old_mu_plugins.php` - Removes deprecated mu-plugins from versions < 3.4.4
- `remove_mu_plugins_batcache_on_uninstall.php` - Cleanup logic for uninstall
- `uninstall.php` - Comprehensive cleanup of all plugin options and files on uninstall

## Development Commands

### Code Quality
```bash
# Run PHP CodeSniffer (WordPress coding standards)
phpcs --standard=.phpcs.xml .

# Fix auto-fixable issues
phpcbf --standard=.phpcs.xml .
```

### Testing
This plugin requires a Pressable hosting environment for full functionality. Local testing should mock the following:
- `IS_PRESSABLE` constant (defined by Pressable)
- `Edge_Cache_Plugin` class (provided by Pressable mu-plugin)
- Batcache configuration

## Important Notes for Development

### Security Patterns
- All AJAX/form actions use WordPress nonce verification
- User capability checks use `manage_options`, `edit_posts`, or `manage_woocommerce` depending on action
- Direct file access is blocked with `if (!defined('ABSPATH')) exit;`
- Input sanitization via `esc_html()`, `esc_url()`, and validation callbacks

### Permission Checks
Admin bar flush cache button is visible to:
- Administrators
- Editors
- Shop managers (WooCommerce users with `manage_woocommerce` capability)

### File Writing Operations
When creating or modifying files:
1. Use `wp_mkdir_p()` for directory creation
2. Check file existence before operations
3. Use `copy()` for mu-plugin creation
4. Call `wp_cache_flush()` after mu-plugin changes to ensure immediate effect
5. Use `wp_opcache_invalidate()` after modifying wp-config.php

### Option Naming Convention
Plugin options use these naming patterns:
- Main settings: `pressable_cache_management_options`
- Feature-specific: `edge_cache_settings_tab_options`, `remove_pressable_branding_tab_options`
- Timestamps: `*-time-stamp` (e.g., `edge-cache-purge-time-stamp`)
- Notices: `*_activate_notice`
- Status flags: `edge-cache-enabled`, `edge-cache-status`

### Admin Notices
Admin notices should:
- Check current screen with `get_current_screen()`
- Only display on `toplevel_page_pressable_cache_management` screen ID
- Use transient storage for temporary notices where appropriate
- Use `is-dismissible` class for user-closeable notices

### Edge Cache Best Practices
When working with Edge Cache:
1. Always check if `Edge_Cache_Plugin` class exists
2. Get instance via `Edge_Cache_Plugin::get_instance()`
3. Check status before purge operations
4. Handle `WP_Error` returns from Edge Cache methods
5. Update option `edge-cache-purge-time-stamp` with UTC timestamp after successful purges

### Localization
- All user-facing strings use `__()` or `esc_html__()` with `pressable_cache_management` text domain
- Translation files go in `languages/` directory
- Plugin loads text domain on `plugins_loaded` hook

## File Structure Quick Reference

```
pressable-cache-management/
├── admin/
│   ├── custom-functions/     # Feature implementations
│   ├── admin-menu.php         # Menu registration
│   ├── settings-page.php      # UI rendering
│   ├── settings-register.php  # Settings API registration
│   ├── settings-callbacks.php # Form field callbacks
│   └── settings-validate.php  # Input validation
├── languages/                 # Translation files
├── pressable-cache-management.php  # Main plugin file
├── uninstall.php             # Cleanup on uninstall
├── remove_old_mu_plugins.php # Version migration cleanup
└── .phpcs.xml                # PHP CodeSniffer rules
```
