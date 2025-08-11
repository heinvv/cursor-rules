# Playwright Widget Testing Rules

## Overview
This guide provides systematic rules for creating comprehensive Playwright tests for Elementor widgets, including advanced conditional control testing. 

**Default Approach**: Use randomized configuration tests with 3 test variations plus a default widget test (16 total screenshots) for efficient, comprehensive coverage including responsive testing. Individual control testing is optional and should be skipped by default using `test.skip()`.

### Helper functions first
- Before writing or expanding tests, ensure required helper functions exist in `tests/playwright/pages/editor-page.ts` and are adequate.
- Prefer adding a new, narrowly scoped helper over modifying an existing one.
- Only update existing helpers when strictly necessary. Provide a clear reason (e.g., DOM changed, selector is flaky) and reference a failing test or recorded step.

## 🎯 Core Testing Philosophy

### 1. **Discover Before Testing**
- **Never assume control values** - always investigate actual options first
- Create temporary investigation tests to discover control structure
- Record actual values, icon classes, and control types before writing tests
- **Test conditional controls** - many controls are hidden until parent switches are enabled
- Delete investigation tests after gathering required information

### Use the Playwright recorder for selectors
- Run tests headed and pause where discovery is needed: insert `await page.pause()` or launch with the debug UI.
- Use the recorder to click through the Elementor UI and copy robust selectors (`getByRole`, `getByText`, `locator().filter({ hasText })`).
- Prefer stable, semantic, built-in Playwright selectors, such as `getByRole`, `getByText`, `getByLabel` etc, over brittle CSS paths. For example, in the Icons Manager:
  - `await page.locator('div').filter({ hasText: /^Address book$/ }).first().click();`
  - `await page.getByRole('button', { name: 'Insert' }).click();`
  - Use `await page.locator('div').filter({ hasText: /^Address book$/ }).first().waitFor();` before clicking when needed.

### 2. **Test ALL Controls, Including Conditional Ones**
- Many controls are hidden inside collapsible sections AND behind conditional logic
- Use systematic discovery to find controls in all style sections
- **Test switcher-dependent controls** by enabling parent switchers first
- Test approximately 20% of available options for each control
- Focus on actual widget impact, not just panel interactions

### 3. **No Assumptions About Control Types or Visibility**
- Choose controls use icon classes (e.g., `eicon-order-start`)
- Select controls use actual option values (e.g., `50%`, `center center`)
- Slider controls use numeric strings (e.g., `'100'`, not `100`)
- Switcher controls use boolean values (`true`/`false`)
- **Conditional controls may be hidden** with `.elementor-hidden-control` class

## 🔍 Discovery Process

### Step 1: Initial Widget Investigation
```typescript
test.only( 'Investigate widget controls', async () => {
    await editor.addWidget( 'widget-name' );
    await editor.waitForPanelToLoad();
    await editor.setWidgetTab( 'style' );
    
    // Check all visible controls first
    const allControls = await page.locator( '.elementor-control:visible' ).all();
    console.log( `Total visible controls: ${allControls.length}` );
    
    // Document each control's structure
    for ( const control of allControls ) {
        const controlClasses = await control.getAttribute( 'class' );
        const dataSetting = await control.getAttribute( 'data-setting' );
        console.log( `Control: ${dataSetting}, Classes: ${controlClasses}` );
    }
});
```

### Step 2: Section Discovery
```typescript
// Systematically open each section to reveal hidden controls
const styleSections = [
    'section_name_1',
    'section_name_2',
    // ... discover by inspecting the widget
];

for ( const sectionName of styleSections ) {
    await editor.openSection( sectionName );
    await page.waitForTimeout( 500 );
    
    // Document new controls that appear
    const newControls = await page.locator( '.elementor-control:visible' ).all();
    // ... log control details
    
    await editor.closeSection( sectionName );
}
```

### Step 3: Conditional Controls Discovery
```typescript
// Test switcher controls that reveal additional controls
const switcherControls = [ 'has_alternate_padding', 'show_alternate_background' ];

for ( const switcherName of switcherControls ) {
    console.log( `\n--- Testing ${switcherName} conditional controls ---` );
    
    // Test OFF state
    await editor.setSwitcherControlValue( switcherName, false );
    await page.waitForTimeout( 500 );
    
    const hiddenControls = await page.locator( 
        `.elementor-control[class*="${switcherName.replace('has_', '').replace('show_', '')}"].elementor-hidden-control` 
    ).count();
    console.log( `Hidden controls when switcher OFF: ${hiddenControls}` );
    
    // Test ON state
    await editor.setSwitcherControlValue( switcherName, true );
    await page.waitForTimeout( 500 );
    
    const visibleControls = await page.locator( 
        `.elementor-control[class*="${switcherName.replace('has_', '').replace('show_', '')}"]:not(.elementor-hidden-control)` 
    ).count();
    console.log( `Visible controls when switcher ON: ${visibleControls}` );
    
    // Document revealed controls
    const revealedControls = await page.locator( 
        `.elementor-control[class*="${switcherName.replace('has_', '').replace('show_', '')}"]:not(.elementor-hidden-control)` 
    ).all();
    
    for ( const control of revealedControls ) {
        const controlClasses = await control.getAttribute( 'class' );
        const dataSetting = await control.getAttribute( 'data-setting' );
        console.log( `  Revealed control: ${dataSetting}` );
        console.log( `    Classes: ${controlClasses}` );
    }
}
```

### Step 4: Nested Conditional Logic Discovery
```typescript
// Test controls that depend on other control values
const backgroundTypeControl = page.locator( '.elementor-control-alternate_background_background' );
if ( await backgroundTypeControl.isVisible() ) {
    // Test different background types
    await editor.setChooseControlValue( 'alternate_background_background', 'eicon-paint-brush' );
    await page.waitForTimeout( 500 );
    
    const colorControl = page.locator( '.elementor-control-alternate_background_color' );
    const isColorVisible = await colorControl.isVisible() && 
        !await colorControl.getAttribute( 'class' ).then( classes => classes?.includes( 'elementor-hidden-control' ) );
    console.log( `Color control visible with classic background: ${isColorVisible}` );
    
    await editor.setChooseControlValue( 'alternate_background_background', 'eicon-barcode' );
    await page.waitForTimeout( 500 );
    
    const gradientNotice = page.locator( '.elementor-control-alternate_background_gradient_notice' );
    const isGradientNoticeVisible = await gradientNotice.isVisible() && 
        !await gradientNotice.getAttribute( 'class' ).then( classes => classes?.includes( 'elementor-hidden-control' ) );
    console.log( `Gradient notice visible with gradient background: ${isGradientNoticeVisible}` );
}
```

## 🧪 Test Structure Rules

### 1. **File Organization**
- One main test file per widget: `widget-name.test.ts`
- Group related controls into logical test functions
- **Include conditional control tests** as separate test functions
- Always include a basic "widget can be added" test
- Use descriptive test names that indicate what's being tested

### 2. **Test Function Categories (Default Approach)**
```typescript
// Basic functionality
test( 'Widget can be added to the page', async () => { ... });

// DEFAULT: Randomized configuration tests (3 tests, includes all control types)
for ( let loopIndex = 0; loopIndex < 3; loopIndex++ ) {
    test( `Widget randomized configuration test ${ loopIndex + 1 }`, async () => { ... });
}

// OPTIONAL: Individual control tests (use test.skip by default)
test.skip( 'Widget Content tab controls - OPTIONAL', async () => { ... });
test.skip( 'Widget direction and alignment controls - OPTIONAL', async () => { ... });
test.skip( 'Widget image controls - OPTIONAL', async () => { ... });
test.skip( 'Widget background and layout controls - OPTIONAL', async () => { ... });

// DISCOVERY: Only during development (delete after)
test.skip( 'Widget conditional controls discovery - DELETE AFTER USE', async () => { ... });
```

**Randomized Test Benefits:**
- Tests multiple control combinations in realistic scenarios
- Includes conditional control testing progressively
- Reduces screenshot count from 20-50+ to just 6
- Covers edge cases through value cycling
- Maintains comprehensive test coverage

### 3. **Conditional Control Testing Patterns**

#### Testing Switcher-Dependent Controls
```typescript
// Enable parent switcher first
await editor.setSwitcherControlValue( 'has_alternate_padding', true );
await page.waitForTimeout( 500 );

// Test revealed controls
const paddingControl = page.locator( '.elementor-control-alternate_padding_horizontal' );
const isVisible = await paddingControl.isVisible() && 
    !await paddingControl.getAttribute( 'class' ).then( classes => classes?.includes( 'elementor-hidden-control' ) );

if ( isVisible ) {
    const sliderValues = [ '10', '25' ];
    for ( const value of sliderValues ) {
        await editor.setSliderControlValue( 'alternate_padding_horizontal', value );
        await expect.soft( widget ).toHaveScreenshot( `widget-padding-${value}-editor.png` );
    }
}
```

#### Testing Nested Conditional Logic
```typescript
// Enable primary condition
await editor.setSwitcherControlValue( 'show_alternate_background', true );
await page.waitForTimeout( 500 );

// Test secondary conditions
const backgroundTypes = [ 'eicon-paint-brush', 'eicon-barcode' ];
for ( const bgType of backgroundTypes ) {
    const bgTypeControl = page.locator( '.elementor-control-alternate_background_background' );
    const isBgTypeVisible = await bgTypeControl.isVisible() && 
        !await bgTypeControl.getAttribute( 'class' ).then( classes => classes?.includes( 'elementor-hidden-control' ) );
        
    if ( isBgTypeVisible ) {
        await editor.setChooseControlValue( 'alternate_background_background', bgType );
        await page.waitForTimeout( 500 );
        await expect.soft( widget ).toHaveScreenshot( `widget-bg-${bgType.replace('eicon-', '')}-editor.png` );
    }
}
```

#### Choose Controls
```typescript
const controlOptions = [ 'eicon-icon-1', 'eicon-icon-2' ];
for ( const option of controlOptions ) {
    await editor.setChooseControlValue( 'control_name', option );
    await expect.soft( widget ).toHaveScreenshot( `widget-control-${option.replace('eicon-', '')}-editor.png` );
}
```

#### Select Controls
```typescript
const selectOptions = [ 'value1', 'value2', 'value3' ];
for ( const option of selectOptions ) {
    await editor.setSelectControlValue( 'control_name', option );
    await expect.soft( widget ).toHaveScreenshot( `widget-control-${option}-editor.png` );
}
```

#### Slider Controls
```typescript
const sliderValues = [ '10', '50', '100' ];
for ( const value of sliderValues ) {
    await editor.setSliderControlValue( 'control_name', value );
    await expect.soft( widget ).toHaveScreenshot( `widget-control-${value}-editor.png` );
}
```

#### Switcher Controls
### Helper change policy
- Avoid modifying helpers unless there is a verified need (broken due to DOM change, persistent flakiness, or missing capability).
- When a change is needed, first attempt to add a new helper method with a narrow purpose to avoid regressions.
- Validate changes using a headed run and, where applicable, by recording the interaction with the Playwright recorder to confirm selectors.

```typescript
const switcherControls = [ 'control_1', 'control_2' ];
for ( const controlName of switcherControls ) {
    await editor.setSwitcherControlValue( controlName, true );
    await expect.soft( widget ).toHaveScreenshot( `widget-${controlName}-on-editor.png` );
    
    await editor.setSwitcherControlValue( controlName, false );
    await expect.soft( widget ).toHaveScreenshot( `widget-${controlName}-off-editor.png` );
}
```

## 📸 Screenshot Rules

### 1. **Screenshot Strategy (Default: Randomized)**

#### **Randomized Screenshots (Recommended Default)**
Use randomized configuration tests with minimal screenshots for efficient, comprehensive coverage:

```typescript
const CONTROL_VALUES = {
    direction: [ 'eicon-order-start', 'eicon-order-end' ],
    alignment: [ 'eicon-align-start-v', 'eicon-align-center-v', 'eicon-align-end-v' ],
    backgroundColors: [ '#FF5722', '#2196F3', '#4CAF50' ],
};

// Create 3 randomized configuration tests with cycling values
for ( let loopIndex = 0; loopIndex < 3; loopIndex++ ) {
    test( `Widget randomized configuration test ${ loopIndex + 1 }`, async () => {
        await editor.addWidget( 'widget-name' );
        
        // Apply different control combinations using modulo cycling
        await editor.setChooseControlValue( 'direction', editor.getControlValueByIndex( CONTROL_VALUES.direction, loopIndex ) );
        await editor.setChooseControlValue( 'alignment', editor.getControlValueByIndex( CONTROL_VALUES.alignment, loopIndex ) );
        
        // Test conditional controls progressively
        if ( loopIndex >= 1 ) {
            await editor.setSwitcherControlValue( 'has_feature', true );
            // Test revealed controls
        }
        
        if ( loopIndex >= 2 ) {
            await editor.setSwitcherControlValue( 'show_background', true );
            // Test additional revealed controls
        }
        
        // Take 1 editor + 3 responsive frontend screenshots per configuration
        await expect.soft( editorWidget ).toHaveScreenshot( `widget-config-${ loopIndex + 1 }-editor.png` );
        
        await editor.publishAndViewPage();

        await expect.soft( frontendWidget ).toHaveScreenshot( `widget-config-${ loopIndex + 1 }-desktop-frontend.png` );
        
        // Tablet
        await page.setViewportSize( viewportSize.tablet );
        await expect.soft( frontendWidget ).toHaveScreenshot( `widget-config-${ loopIndex + 1 }-tablet-frontend.png` );
        
        // Mobile
        await page.setViewportSize( viewportSize.mobile );
        await expect.soft( frontendWidget ).toHaveScreenshot( `widget-config-${ loopIndex + 1 }-mobile-frontend.png` );
    } );
}
```

**Benefits of Randomized Approach:**
- **Efficient**: Only 12 total screenshots (3 editor + 9 responsive frontend) instead of dozens
- **Comprehensive**: Tests multiple control combinations and conditional states
- **Responsive**: Tests desktop, tablet, and mobile layouts automatically
- **Manageable**: Easier to maintain and update
- **Realistic**: Tests real-world usage patterns with multiple controls configured

#### **Individual Control Screenshots (Optional)**
Use for detailed control analysis or when specific control variations need documentation:

```typescript
// Optional: Individual control testing for detailed analysis
test.skip( 'Widget individual control variations - OPTIONAL', async () => {
    const chooseOptions = [ 'eicon-option1', 'eicon-option2', 'eicon-option3' ];
    for ( const option of chooseOptions ) {
        await editor.setChooseControlValue( 'control_name', option );
        await expect.soft( widget ).toHaveScreenshot( `widget-control-${option.replace('eicon-', '')}-editor.png` );
    }
    
    const sliderValues = [ '10', '50', '100' ];
    for ( const value of sliderValues ) {
        await editor.setSliderControlValue( 'slider_control', value );
        await expect.soft( widget ).toHaveScreenshot( `widget-slider-${value}-editor.png` );
    }
});
```

**When to Use Individual Control Screenshots:**
- Widget documentation needs detailed control examples
- Debugging specific control issues
- Creating comprehensive control reference
- QA requires individual control validation

### 2. **Naming Convention**

#### **Randomized Screenshots (Default)**
- **Editor screenshots**: `widget-config-{1-3}-editor.png`
- **Desktop frontend screenshots**: `widget-config-{1-3}-desktop-frontend.png`
- **Tablet frontend screenshots**: `widget-config-{1-3}-tablet-frontend.png`
- **Mobile frontend screenshots**: `widget-config-{1-3}-mobile-frontend.png`
- **Default widget screenshot**: `widget-default-editor.png`

#### **Individual Control Screenshots (Optional)**
- **Control variations**: `widget-control-value-editor.png`
- **Conditional controls**: `widget-conditional-state-editor.png`
- **Responsive controls**: `widget-control-value-tablet-editor.png`

### 3. **Screenshot Target**
```typescript
// Always target the widget, not the panel
const previewFrame = editor.getPreviewFrame();
const widget = previewFrame.locator( '.widget-css-class' );
await expect.soft( widget ).toHaveScreenshot( 'filename.png' );
```

### 4. **Efficient Screenshot Management**
- **Default**: Use randomized tests with 1 editor + 3 responsive frontend screenshots per configuration
- **Default widget**: Always capture default state before any styling
- **Optional**: Add individual control tests only when specifically needed
- **Capture conditional states** progressively across randomized tests
- **Responsive testing**: Include desktop, tablet, and mobile for all frontend screenshots
- Use `expect.soft()` to continue testing even if screenshots fail


```typescript
test.beforeEach( async ( { browser, apiRequests }, testInfo ) => {
    context = await browser.newContext();
    page = await context.newPage();
    wpAdmin = new WpAdminPage( page, testInfo, apiRequests );
    editor = await wpAdmin.openNewPage();
    await editor.closeNavigatorIfOpen();
} );

test.afterEach( async () => {
    await context.close();
} );
```

## 🚀 Frontend Testing Rules

### 1. **Editor vs Frontend Comparison**
```typescript
// Configure in editor (including conditional controls)
await editor.setWidgetTab( 'style' );
await editor.setChooseControlValue( 'control_1', 'value1' );
await editor.setSwitcherControlValue( 'conditional_control', true );
await editor.setSliderControlValue( 'revealed_control', '100' );

// Screenshot editor state
await expect.soft( editorWidget ).toHaveScreenshot( 'widget-configured-editor.png' );

// Publish and view frontend
await editor.publishAndViewPage();

const frontendWidget = page.locator( '.widget-css-class' );
await expect( frontendWidget ).toBeVisible();

await expect.soft( frontendWidget ).toHaveScreenshot( 'widget-configured-desktop-frontend.png' );

await page.setViewportSize( viewportSize.tablet );
await expect.soft( frontendWidget ).toHaveScreenshot( 'widget-configured-tablet-frontend.png' );

await page.setViewportSize( viewportSize.mobile );
await expect.soft( frontendWidget ).toHaveScreenshot( 'widget-configured-mobile-frontend.png' );
```

### 2. **Responsive Testing**
All frontend screenshots should include responsive testing by default:

```typescript
import { viewportSize } from '../../../../enums/viewport-sizes';

const frontendWidget = page.locator( '.widget-css-class' );

// Desktop (default viewport)
await expect.soft( frontendWidget ).toHaveScreenshot( 'widget-desktop-frontend.png' );

// Tablet
await page.setViewportSize( viewportSize.tablet );
await expect.soft( frontendWidget ).toHaveScreenshot( 'widget-tablet-frontend.png' );

// Mobile
await page.setViewportSize( viewportSize.mobile );
await expect.soft( frontendWidget ).toHaveScreenshot( 'widget-mobile-frontend.png' );
```

**Note**: This responsive pattern is included by default in all randomized configuration tests.

## ⚡ Performance Rules

### 1. **Efficient Control Testing**
- Use loops to test multiple values for the same control type
- Group related controls in the same test function
- Open/close sections only when necessary
- Use named constants for delays instead of magic numbers
- **Wait for conditional controls** to appear/disappear before testing

### 2. **Avoid Magic Numbers**
- **Never use hardcoded values** without explanation
- Use predefined constants from `viewportSize` enum for responsive testing
- Define named constants for delays, timeouts, and control values
- Use existing constants from the Elementor framework when available

```typescript
// Good: Using predefined viewport sizes
import { viewportSize } from '../../../../enums/viewport-sizes';
await page.setViewportSize( viewportSize.tablet );

// Good: Named constants for clarity
const CONDITIONAL_CONTROL_DELAY = 500;
const INTERACTION_TIMEOUT = 1000;
await page.waitForTimeout( CONDITIONAL_CONTROL_DELAY );

// Bad: Magic numbers
await page.setViewportSize( { width: 768, height: 1024 } );
await page.waitForTimeout( 500 );
```

### 3. **Test Organization**
- Test ~20% of available options, not all options
- Focus on visually distinct options
- Prioritize edge cases (min/max values, contrasting styles)
- Use meaningful control values that demonstrate visual differences
- **Test conditional control states** that show significant differences

## 🧹 Code Quality Rules

### 1. **No Error Handling**
- **Never use try/catch blocks** around control interactions
- Let tests fail clearly if controls don't work
- Use proper selectors and wait for elements to be ready
- **Check control visibility** before attempting to interact

### 2. **No Comments or Logs in Final Tests**
- Remove all `console.log` statements from final tests
- Remove all comments (`// ...`) from final tests
- Use descriptive variable and function names instead

### 3. **Clean Code Patterns**
```typescript
// Good: Descriptive and clean with helper functions
const titleOptions = [ 'h2', 'h4', 'div' ];
for ( let loopIndex = 0; loopIndex < 3; loopIndex++ ) {
    await editor.setSelectControlValue( 'title_tag', editor.getControlValueByIndex( titleOptions, loopIndex ) );
    await expect.soft( widget ).toHaveScreenshot( `widget-title-config-${loopIndex + 1}-editor.png` );
}

// Good: Background control with helper function
await editor.setBackgroundColorControlValue( 'background_background', 'background_color', '#FF5722' );

// Good: Conditional control testing
const paddingControl = page.locator( '.elementor-control-alternate_padding_horizontal' );
const isVisible = await paddingControl.isVisible() && 
    !await paddingControl.getAttribute( 'class' ).then( classes => classes?.includes( 'elementor-hidden-control' ) );
    
if ( isVisible ) {
    await editor.setSliderControlValue( 'alternate_padding_horizontal', '25' );
}

// Good: Frontend navigation with helper function
await editor.publishAndViewPage();

// Bad: With comments and logs
try {
    // Setting title tag to h2
    await editor.setSelectControlValue( 'title_tag', 'h2' );
    console.log( 'Set title tag to h2' );
} catch ( error ) {
    console.log( 'Failed to set title tag' );
}
```

### 4. **Available Helper Functions**

These helper functions are available in `editor-page.ts` for consistent testing:

```typescript
// Value cycling for randomized tests
editor.getControlValueByIndex( controlValues: any[], loopIndex: number ): any

// Background color control with visibility check
editor.setBackgroundColorControlValue( backgroundControlId: string, colorControlId: string, colorValue: string ): Promise<void>

// Frontend navigation
editor.publishAndViewPage(): Promise<void>
```

**Usage Examples:**
```typescript
// Cycle through control values
const directions = [ 'eicon-order-start', 'eicon-order-end' ];
await editor.setChooseControlValue( 'direction', editor.getControlValueByIndex( directions, loopIndex ) );

// Set background color safely
await editor.setBackgroundColorControlValue( 'background_background', 'background_color', '#FF5722' );

// Navigate to published page
await editor.publishAndViewPage();
```

## 🎯 Control Discovery Checklist

When testing a new widget, systematically check for these control types:

### Content Tab Controls
- [ ] Text content controls (text, textarea)
- [ ] Media controls (image, video selectors)
- [ ] URL/link controls
- [ ] Select dropdowns (tag types, etc.)
- [ ] Repeater controls (lists, galleries)

### Style Tab Controls
- [ ] **Layout Section**: Direction, alignment, positioning
- [ ] **Typography Section**: Font controls, text styling
- [ ] **Colors Section**: Text colors, background colors
- [ ] **Background Section**: Background types (classic, gradient)
- [ ] **Border Section**: Border styling, radius
- [ ] **Spacing Section**: Padding, margin, gaps
- [ ] **Image Section**: Image styling, shapes, filters
- [ ] **Button/CTA Section**: Button styling, spacing
- [ ] **Effects Section**: Shadows, transforms
- [ ] **Advanced Section**: Custom CSS, animations
- [ ] **Responsive Section**: Device-specific overrides
- [ ] **Alternate Section**: Alternative styling options

### Conditional Controls Checklist
- [ ] **Switcher-dependent controls**: Controls revealed by enabling switchers
- [ ] **Choose-dependent controls**: Controls that appear based on choose control values
- [ ] **Select-dependent controls**: Controls that appear based on select option values
- [ ] **Nested conditions**: Multi-level conditional logic
- [ ] **Responsive variants**: Controls with desktop/tablet/mobile variants
- [ ] **Group controls**: Controls that belong to control groups (background, typography, etc.)

### Advanced Tab Controls
- [ ] Motion effects
- [ ] Custom CSS
- [ ] Custom attributes
- [ ] Visibility conditions

## 🔍 Conditional Control Detection Patterns

### Common Conditional Control Patterns
```typescript
// Pattern 1: Switcher reveals controls
'has_*' or 'show_*' switcher → reveals '*' related controls

// Pattern 2: Background type reveals specific controls
'background_background' choose → reveals type-specific controls
- 'classic' → color, image controls
- 'gradient' → gradient controls, notices

// Pattern 3: Responsive variants
'control_name' → 'control_name_tablet', 'control_name_mobile'

// Pattern 4: Group controls
'group_*' → multiple related controls with same prefix
```

### Detection Methods
```typescript
// Method 1: Count hidden vs visible controls
const hiddenCount = await page.locator( '.elementor-control[class*="keyword"].elementor-hidden-control' ).count();
const visibleCount = await page.locator( '.elementor-control[class*="keyword"]:not(.elementor-hidden-control)' ).count();

// Method 2: Check individual control visibility
const isControlVisible = await control.isVisible() && 
    !await control.getAttribute( 'class' ).then( classes => classes?.includes( 'elementor-hidden-control' ) );

// Method 3: Before/after switcher toggle comparison
await editor.setSwitcherControlValue( 'switcher_name', false );
const beforeCount = await page.locator( '.elementor-control:visible' ).count();

await editor.setSwitcherControlValue( 'switcher_name', true );
const afterCount = await page.locator( '.elementor-control:visible' ).count();

console.log( `Controls revealed: ${afterCount - beforeCount}` );
```

## 🚀 Quick Start Template

```typescript
import { expect } from '@playwright/test';
import { parallelTest as test } from '../../../../parallelTest';
import WpAdminPage from '../../../../pages/wp-admin-page';
import { viewportSize } from '../../../../enums/viewport-sizes';

const WIDGET_CSS_CLASS = '.widget-css-class';

const CONTROL_VALUES = {
    direction: [ 'eicon-order-start', 'eicon-order-end' ],
    alignment: [ 'eicon-align-start-v', 'eicon-align-center-v', 'eicon-align-end-v' ],
    switcherControls: [ 'has_feature', 'show_feature' ],
    backgroundColors: [ '#FF5722', '#2196F3', '#4CAF50' ],
    sliderValues: [ '10', '25', '40' ],
};

let context, page, wpAdmin, editor;

test.beforeEach( async ( { browser, apiRequests }, testInfo ) => {
    context = await browser.newContext();
    page = await context.newPage();
    wpAdmin = new WpAdminPage( page, testInfo, apiRequests );
    editor = await wpAdmin.openNewPage();
    await editor.closeNavigatorIfOpen();
} );

test.afterEach( async () => {
    await context.close();
} );

test( 'WIDGET_NAME widget can be added to the page', async () => {
	await editor.addWidget( 'WIDGET_SLUG' );
	
	const previewFrame = editor.getPreviewFrame();
	const widget = previewFrame.locator( WIDGET_CSS_CLASS );
	await expect( widget ).toBeVisible();
	await expect.soft( widget ).toHaveScreenshot( 'widget-default-editor.png' );
	
	await editor.publishAndViewPage();
	
	const frontendWidget = page.locator( WIDGET_CSS_CLASS );
	await expect( frontendWidget ).toBeVisible();

	await expect.soft( frontendWidget ).toHaveScreenshot( 'widget-default-desktop-frontend.png' );
	
	await page.setViewportSize( viewportSize.tablet );
	await expect.soft( frontendWidget ).toHaveScreenshot( 'widget-default-tablet-frontend.png' );
	
	await page.setViewportSize( viewportSize.mobile );
	await expect.soft( frontendWidget ).toHaveScreenshot( 'widget-default-mobile-frontend.png' );
} );

// DEFAULT: Randomized configuration tests (3 tests, 6 screenshots total)
for ( let loopIndex = 0; loopIndex < 3; loopIndex++ ) {
    test( `WIDGET_NAME randomized configuration test ${ loopIndex + 1 }`, async () => {
        await editor.addWidget( 'WIDGET_SLUG' );
        await editor.waitForPanelToLoad();
        
        const previewFrame = editor.getPreviewFrame();
        const editorWidget = previewFrame.locator( WIDGET_CSS_CLASS );
        await expect( editorWidget ).toBeVisible();
        
        // Apply randomized control combinations
        await editor.setWidgetTab( 'style' );
        await editor.setChooseControlValue( 'direction', editor.getControlValueByIndex( CONTROL_VALUES.direction, loopIndex ) );
        await editor.setChooseControlValue( 'alignment', editor.getControlValueByIndex( CONTROL_VALUES.alignment, loopIndex ) );
        await editor.setSliderControlValue( 'slider_control', editor.getControlValueByIndex( CONTROL_VALUES.sliderValues, loopIndex ) );
        
        // Test conditional controls progressively
        if ( loopIndex >= 1 ) {
            await editor.setSwitcherControlValue( 'has_feature', true );
            await page.waitForTimeout( 500 );
            
            const conditionalControl = page.locator( '.elementor-control-feature_control' );
            const isVisible = await conditionalControl.isVisible() && 
                ! await conditionalControl.getAttribute( 'class' ).then( ( classes ) => classes?.includes( 'elementor-hidden-control' ) );
            if ( isVisible ) {
                await editor.setSliderControlValue( 'feature_control', editor.getControlValueByIndex( CONTROL_VALUES.sliderValues, loopIndex ) );
            }
        }
        
        if ( loopIndex >= 2 ) {
            await editor.setSwitcherControlValue( 'show_background', true );
            await page.waitForTimeout( 500 );
            await editor.setBackgroundColorControlValue( 'background_background', 'background_color', editor.getControlValueByIndex( CONTROL_VALUES.backgroundColors, loopIndex ) );
        }
        
        await expect.soft( editorWidget ).toHaveScreenshot( `widget-config-${ loopIndex + 1 }-editor.png` );
        
        await editor.publishAndViewPage();
        
        const frontendWidget = page.locator( WIDGET_CSS_CLASS );
        await expect( frontendWidget ).toBeVisible();

        await expect.soft( frontendWidget ).toHaveScreenshot( `widget-config-${ loopIndex + 1 }-desktop-frontend.png` );
        
        await page.setViewportSize( viewportSize.tablet );
        await expect.soft( frontendWidget ).toHaveScreenshot( `widget-config-${ loopIndex + 1 }-tablet-frontend.png` );
        
        await page.setViewportSize( viewportSize.mobile );
        await expect.soft( frontendWidget ).toHaveScreenshot( `widget-config-${ loopIndex + 1 }-mobile-frontend.png` );
    } );
}

// OPTIONAL: Individual control tests (use test.skip by default)
test.skip( 'WIDGET_NAME individual control variations - OPTIONAL', async () => {
    await editor.addWidget( 'WIDGET_SLUG' );
    
    const previewFrame = editor.getPreviewFrame();
    const widget = previewFrame.locator( WIDGET_CSS_CLASS );
    
    // Individual control testing
    for ( const option of CONTROL_VALUES.direction ) {
        await editor.setChooseControlValue( 'direction', option );
        await expect.soft( widget ).toHaveScreenshot( `widget-direction-${option.replace('eicon-', '')}-editor.png` );
    }
} );
```

## 🏁 Testing Workflow

1. **Discovery Phase** (temporary tests)
   - Investigate control structure
   - **Discover conditional controls** by testing all switchers
   - Document all available controls and their dependencies
   - Record actual values and types
   - Delete discovery tests

2. **Default Implementation Phase (Randomized)**
   - Create basic widget test (default state)
   - **Implement 3 randomized configuration tests**
   - Use `editor.getControlValueByIndex()` for value cycling
   - Test conditional controls progressively (loopIndex >= 1, >= 2)
   - Generate only 6 screenshots total (3 editor + 3 frontend)
   - Use descriptive `CONTROL_VALUES` constants

3. **Optional Individual Control Phase**
   - Add individual control tests only if specifically needed
   - Use `test.skip()` by default to keep them available but not running
   - Generate individual screenshots for detailed documentation
   - Enable only for debugging or comprehensive control analysis

4. **Frontend Testing Phase**
   - Included in randomized tests by default
   - Test editor vs frontend for each configuration
   - Add responsive testing if needed
   - Use `editor.publishAndViewPage()` for navigation

5. **Cleanup Phase**
   - Remove all comments and logs
   - Remove try/catch blocks
   - Verify naming conventions
   - Extract helper functions to `editor-page.ts`
   - Run final test suite

## 🎯 Default Approach Summary

**Recommended Pattern**: 
- 1 basic test (default state): 4 screenshots (1 editor + 3 responsive frontend)
- 3 randomized configuration tests: 12 screenshots (3 editor + 9 responsive frontend)
- **Total: 16 screenshots** instead of 50-150+ individual control screenshots

**Screenshot Breakdown**:
- **Default widget**: 1 editor + 3 responsive frontend (desktop, tablet, mobile)
- **Configuration 1**: 1 editor + 3 responsive frontend
- **Configuration 2**: 1 editor + 3 responsive frontend  
- **Configuration 3**: 1 editor + 3 responsive frontend

**Optional Enhancement**:
- Add individual control tests using `test.skip()` for future reference
- Enable specific tests only when detailed analysis is needed

Following these rules will result in comprehensive, maintainable, and efficient widget tests that provide excellent coverage of all widget functionality, including complex conditional control logic and responsive behavior, while maintaining reasonable screenshot counts and test execution times.
