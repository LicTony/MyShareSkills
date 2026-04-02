---
name: flutter-building-layouts
description: Builds Flutter layouts using the constraint system and modern layout widgets. Use when creating or refining the UI structure of a Flutter application.
metadata:
  model: models/gemini-3.1-pro-preview
  last_modified: Thu, 2 Apr 2026 00:00:00 GMT

---
# Architecting Flutter Layouts

> **Version:** 1.0.0 — 2026-04-02

## Contents
- [Core Layout Principles](#core-layout-principles)
- [Structural Widgets](#structural-widgets)
- [Advanced Layout Widgets](#advanced-layout-widgets)
- [Adaptive and Responsive Design](#adaptive-and-responsive-design)
- [Scrollable Layouts and Slivers](#scrollable-layouts-and-slivers)
- [Workflow: Implementing a Complex Layout](#workflow-implementing-a-complex-layout)
- [Examples](#examples)

## Core Layout Principles

Master the fundamental Flutter layout rule: **Constraints go down. Sizes go up. Parent sets position.**

*   **Pass Constraints Down:** Always pass constraints (minimum/maximum width and height) from the parent Widget to its children. A Widget cannot choose its own size independently of its parent's constraints.
*   **Pass Sizes Up:** Calculate the child Widget's desired size within the given constraints and pass this size back up to the parent.
*   **Set Position via Parent:** Define the `x` and `y` coordinates of a child Widget exclusively within the parent Widget. Children do not know their own position on the screen.
*   **Avoid Unbounded Constraints:** Never pass unbounded constraints (e.g., `double.infinity`) in the cross-axis of a flex box (`Row` or `Column`) or within scrollable regions (`ListView`). This causes render exceptions.
*   **Understand BoxConstraints:** Every widget receives a `BoxConstraints(minWidth, maxWidth, minHeight, maxHeight)` that defines the allowable size range.

## Structural Widgets

Select the appropriate structural Widget based on the required spatial arrangement.

### Basic Layout Widgets

| Widget | Purpose | Key Properties |
|--------|---------|----------------|
| `Row` | Horizontal linear layout | `mainAxisAlignment`, `crossAxisAlignment`, `mainAxisSize` |
| `Column` | Vertical linear layout | `mainAxisAlignment`, `crossAxisAlignment`, `mainAxisSize` |
| `Stack` | Overlapping widgets (Z-axis) | `alignment`, `clipBehavior`, `fit` |
| `Container` | Boxing widget (padding, margin, decoration) | `padding`, `margin`, `decoration`, `constraints` |
| `SizedBox` | Fixed-size box | `width`, `height` |
| `ConstrainedBox` | Explicit constraint wrapper | `constraints: BoxConstraints(...)` |
| `Expanded` | Force child to fill available space | `flex` (default: 1) |
| `Flexible` | Allow child to size up to available space | `flex`, `fit` (FlexFit.tight/loose) |

### When to Use Each

*   **Use `Row` and `Column`:** For linear horizontal or vertical layouts. Control alignment with `mainAxisAlignment` and `crossAxisAlignment`.
*   **Use `Expanded` vs `Flexible`:** 
    - `Expanded` = `Flexible(fit: FlexFit.tight)` - forces child to fill space
    - `Flexible` - allows child to be smaller than available space
*   **Use `Container`:** When you need padding, margins, borders, or background colors in one widget.
*   **Use `Stack`:** For overlapping elements. Use `Positioned` or `Positioned.fill` to anchor children.
*   **Use `SizedBox`:** For fixed dimensions or as a spacer (`SizedBox.shrink()` or `SizedBox(width: 16)`).
*   **Use `ConstrainedBox`:** When you need explicit min/max constraints (more control than `SizedBox`).

## Advanced Layout Widgets

### Constraint and Sizing Widgets

```dart
// ConstrainedBox - Explicit min/max constraints
ConstrainedBox(
  constraints: BoxConstraints(
    minWidth: 100,
    maxWidth: 300,
    minHeight: 50,
    maxHeight: 200,
  ),
  child: ChildWidget(),
)

// AspectRatio - Maintain aspect ratio
AspectRatio(
  aspectRatio: 16 / 9,
  child: VideoPlayer(),
)

// FractionallySizedBox - Percentage-based sizing
FractionallySizedBox(
  widthFactor: 0.5,  // 50% of parent width
  heightFactor: 0.3, // 30% of parent height
  child: ChildWidget(),
)

// FittedBox - Scale child to fit
FittedBox(
  fit: BoxFit.contain,  // or cover, fill, fitWidth, fitHeight, none, scaleDown
  alignment: Alignment.center,
  child: LargeContent(),
)

// IntrinsicWidth/IntrinsicHeight - Size to content (expensive!)
IntrinsicWidth(
  child: Column(children: [...]),
)
```

### Safe Area and Platform Adaptation

```dart
// SafeArea - Avoid notches, status bars, gesture areas
SafeArea(
  top: true,      // Avoid status bar
  bottom: true,   // Avoid home indicator (iOS) / gesture bar (Android)
  left: true,
  right: true,
  minimum: EdgeInsets.all(8),  // Minimum padding
  child: ChildWidget(),
)

// MediaQuery - Access screen metrics
final size = MediaQuery.sizeOf(context);
final padding = MediaQuery.paddingOf(context);  // Safe area insets
final viewInsets = MediaQuery.of(context).viewInsets;  // Keyboard
final textScaler = MediaQuery.of(context).textScaler;  // Font scaling

// Platform-aware layouts
if (Theme.of(context).platform == TargetPlatform.iOS) {
  // iOS-specific styling
} else {
  // Android/Material styling
}
```

## Adaptive and Responsive Design

Apply conditional logic to handle varying screen sizes and form factors.

### Breakpoints (Material Design 3)

```dart
class Breakpoints {
  static const double phone = 0;      // 0-599
  static const double tablet = 600;   // 600-839
  static const double desktop = 840;  // 840-1200
  static const double large = 1200;   // 1200+
}

// Usage in LayoutBuilder
LayoutBuilder(
  builder: (context, constraints) {
    if (constraints.maxWidth >= Breakpoints.desktop) {
      return DesktopLayout();
    } else if (constraints.maxWidth >= Breakpoints.tablet) {
      return TabletLayout();
    } else {
      return MobileLayout();
    }
  },
)
```

### Responsive vs Adaptive

| Approach | When to Use | Tools |
|----------|-------------|-------|
| **Responsive** | Fitting UI into available space | `LayoutBuilder`, `Expanded`, `Flexible`, `FractionallySizedBox` |
| **Adaptive** | Different layouts per form factor | `LayoutBuilder` + conditional rendering, `MediaQuery` |

### LayoutBuilder vs MediaQuery

| Tool | Use When | Rebuilds When |
|------|----------|---------------|
| `LayoutBuilder` | Need parent constraints | Parent size changes |
| `MediaQuery` | Need screen size, orientation, platform | System metrics change |

**Best Practice:** Prefer `LayoutBuilder` for component-level responsiveness. Use `MediaQuery` for app-level adaptations.

### Orientation Handling

```dart
OrientationBuilder(
  builder: (context, orientation) {
    if (orientation == Orientation.portrait) {
      return PortraitLayout();
    } else {
      return LandscapeLayout();
    }
  },
)

// Or with MediaQuery
final orientation = MediaQuery.of(context).orientation;
```

## Scrollable Layouts and Slivers

### Basic Scrollables

```dart
// SingleChildScrollView - Single scrollable child
SingleChildScrollView(
  scrollDirection: Axis.vertical,  // or horizontal
  padding: EdgeInsets.all(16),
  child: Column(children: [...]),
)

// ListView - Scrollable list of children
ListView(
  scrollDirection: Axis.vertical,
  padding: EdgeInsets.all(8),
  children: [...],
)

// ListView.builder - Lazy loading for long lists
ListView.builder(
  itemCount: items.length,
  itemBuilder: (context, index) => ListTile(title: Text(items[index])),
)

// GridView - Grid layout
GridView.count(
  crossAxisCount: 2,
  mainAxisSpacing: 8,
  crossAxisSpacing: 8,
  children: [...],
)

// GridView.builder - Lazy loading grid
GridView.builder(
  gridDelegate: SliverGridDelegateWithFixedCrossAxisCount(
    crossAxisCount: 2,
    mainAxisSpacing: 8,
    crossAxisSpacing: 8,
  ),
  itemCount: items.length,
  itemBuilder: (context, index) => Card(child: Text(items[index])),
)
```

### Slivers (Advanced Scroll Effects)

Slivers are specialized widgets for `CustomScrollView` that enable advanced scroll effects.

```dart
CustomScrollView(
  slivers: [
    // Collapsing app bar
    SliverAppBar(
      title: Text('Title'),
      pinned: true,      // Stay visible at top
      floating: true,    // Show/hide on scroll
      snap: true,        // Snap to show/hide
      expandedHeight: 200,
      flexibleSpace: FlexibleSpaceBar(
        title: Text('Title'),
        background: Image.network('...', fit: BoxFit.cover),
      ),
    ),
    
    // Grid
    SliverGrid(
      gridDelegate: SliverGridDelegateWithFixedCrossAxisCount(
        crossAxisCount: 2,
      ),
      delegate: SliverChildBuilderDelegate(
        (context, index) => Card(child: Text('Item $index')),
        childCount: 20,
      ),
    ),
    
    // List with header
    SliverToBoxAdapter(
      child: Padding(
        padding: EdgeInsets.all(16),
        child: Text('Header'),
      ),
    ),
    
    // Padding in scroll
    SliverPadding(
      padding: EdgeInsets.all(8),
      sliver: SliverList(
        delegate: SliverChildBuilderDelegate(
          (context, index) => ListTile(title: Text('Item $index')),
          childCount: 50,
        ),
      ),
    ),
    
    // Fill remaining space
    SliverFillRemaining(
      child: Center(child: Text('Footer')),
    ),
  ],
)
```

### Common Sliver Patterns

```dart
// Pattern 1: Collapsing header with image
CustomScrollView(
  slivers: [
    SliverAppBar(
      expandedHeight: 250,
      pinned: true,
      flexibleSpace: FlexibleSpaceBar(
        background: Image.network('hero-image.jpg', fit: BoxFit.cover),
      ),
    ),
    SliverList(delegate: SliverChildListDelegate([...]))
  ],
)

// Pattern 2: Tab bar that collapses
CustomScrollView(
  slivers: [
    SliverAppBar(
      title: Text('Title'),
      bottom: TabBar(tabs: [...]),
      pinned: true,
    ),
    SliverFillRemaining(
      child: TabBarView(children: [...]),
    ),
  ],
)

// Pattern 3: NestedScrollView for tabs with scrollable content
NestedScrollView(
  headerSliverBuilder: (context, innerBoxIsScrolled) => [
    SliverOverlapAbsorber(
      handle: NestedScrollView.sliverOverlapAbsorberHandleFor(context),
      sliver: SliverAppBar(title: Text('Title'), pinned: true, floating: true),
    ),
  ],
  body: TabBarView(children: [
    CustomScrollView(
      key: PageStorageKey('tab1'), // Preserve scroll position
      slivers: [
        SliverOverlapInjector(
          handle: NestedScrollView.sliverOverlapAbsorberHandleFor(context),
        ),
        SliverList(delegate: SliverChildBuilderDelegate(...)),
      ],
    ),
    CustomScrollView(
      key: PageStorageKey('tab2'), // Preserve scroll position
      slivers: [
        SliverOverlapInjector(
          handle: NestedScrollView.sliverOverlapAbsorberHandleFor(context),
        ),
        SliverList(delegate: SliverChildBuilderDelegate(...)),
      ],
    ),
  ]),
)
```

## Workflow: Implementing a Complex Layout

Follow this sequential workflow to architect and implement robust Flutter layouts.

### Task Progress
- [ ] **Phase 1: Visual Deconstruction**
  - [ ] Break down the target UI into a hierarchy of rows, columns, and grids.
  - [ ] Identify overlapping elements (requiring `Stack`).
  - [ ] Identify scrolling regions (requiring `ListView`, `SingleChildScrollView`, or `CustomScrollView`).
  - [ ] Identify safe area requirements (notches, gesture bars).
- [ ] **Phase 2: Constraint Planning**
  - [ ] Determine which widgets require tight constraints (fixed size) vs. loose constraints (flexible size).
  - [ ] Identify potential unbounded constraint risks (e.g., `ListView` inside `Column`).
  - [ ] Plan for responsive breakpoints.
- [ ] **Phase 3: Implementation**
  - [ ] Build the layout from the outside in, starting with `Scaffold` and primary structural widgets.
  - [ ] Wrap content in `SafeArea` where needed.
  - [ ] Extract deeply nested layout sections into separate, stateless widgets.
  - [ ] Use `const` constructors wherever possible for performance.
- [ ] **Phase 4: Validation and Feedback Loop**
  - [ ] Run the application on target devices/simulators (mobile, tablet, web).
  - [ ] **Flutter Layout Explorer:** Use the DevTools Layout Explorer to inspect constraints and flex factors.
  - [ ] **Debug Paint:** Enable "Debug Paint" in Flutter Inspector to visualize render boxes.
  - [ ] Check for overflow warnings (yellow/black stripes).
  - [ ] Test orientation changes (portrait/landscape).
  - [ ] Test keyboard appearance (viewInsets).
  - [ ] **Fix overflow:** Wrap in `Expanded` (flex boxes) or scrollable widgets.

## Examples

### Example 1: Resolving Unbounded Constraints

**Anti-pattern:** Placing `ListView` directly inside `Column` causes unbounded height exception.

```dart
// BAD: Throws unbounded height exception
Column(
  children: [
    Text('Header'),
    ListView(
      children: [/* items */],
    ),
  ],
)
```

**Solution:** Wrap in `Expanded` to bound height.

```dart
// GOOD: ListView constrained to remaining space
Column(
  children: [
    Text('Header'),
    Expanded(
      child: ListView(
        children: [/* items */],
      ),
    ),
  ],
)
```

### Example 2: Responsive Card Grid

```dart
class ResponsiveCardGrid extends StatelessWidget {
  final List<CardData> cards;
  
  const ResponsiveCardGrid({required this.cards});
  
  @override
  Widget build(BuildContext context) {
    return LayoutBuilder(
      builder: (context, constraints) {
        int crossAxisCount;
        double childAspectRatio;
        
        if (constraints.maxWidth >= 1200) {
          crossAxisCount = 4;
          childAspectRatio = 16 / 9;
        } else if (constraints.maxWidth >= 800) {
          crossAxisCount = 3;
          childAspectRatio = 16 / 9;
        } else if (constraints.maxWidth >= 400) {
          crossAxisCount = 2;
          childAspectRatio = 1;
        } else {
          crossAxisCount = 1;
          childAspectRatio = 2;
        }
        
        return GridView.builder(
          padding: const EdgeInsets.all(16),
          gridDelegate: SliverGridDelegateWithFixedCrossAxisCount(
            crossAxisCount: crossAxisCount,
            mainAxisSpacing: 16,
            crossAxisSpacing: 16,
            childAspectRatio: childAspectRatio,
          ),
          itemCount: cards.length,
          itemBuilder: (context, index) => CardWidget(data: cards[index]),
        );
      },
    );
  }
}
```

### Example 3: Adaptive Navigation Layout

```dart
class AdaptiveScaffold extends StatelessWidget {
  final Widget body;
  final int selectedIndex;
  final ValueChanged<int> onDestinationSelected;
  
  const AdaptiveScaffold({
    required this.body,
    required this.selectedIndex,
    required this.onDestinationSelected,
  });
  
  @override
  Widget build(BuildContext context) {
    return LayoutBuilder(
      builder: (context, constraints) {
        // Desktop/Tablet: NavigationRail
        if (constraints.maxWidth >= 600) {
          return Scaffold(
            body: Row(
              children: [
                NavigationRail(
                  selectedIndex: selectedIndex,
                  onDestinationSelected: (i) => onDestinationSelected(i),
                  destinations: const [
                    NavigationRailDestination(
                      icon: Icon(Icons.home),
                      label: Text('Home'),
                    ),
                    NavigationRailDestination(
                      icon: Icon(Icons.search),
                      label: Text('Search'),
                    ),
                    NavigationRailDestination(
                      icon: Icon(Icons.settings),
                      label: Text('Settings'),
                    ),
                  ],
                ),
                const VerticalDivider(thickness: 1, width: 1),
                Expanded(child: body),
              ],
            ),
          );
        }
        
        // Mobile: BottomNavigationBar
        return Scaffold(
          body: body,
          bottomNavigationBar: NavigationBar(
            selectedIndex: selectedIndex,
            onDestinationSelected: onDestinationSelected,
            destinations: const [
              NavigationDestination(icon: Icon(Icons.home), label: 'Home'),
              NavigationDestination(icon: Icon(Icons.search), label: 'Search'),
              NavigationDestination(icon: Icon(Icons.settings), label: 'Settings'),
            ],
          ),
        );
      },
    );
  }
}
```

### Example 4: Profile Page with Collapsing Header

```dart
class ProfilePage extends StatelessWidget {
  final UserData user;
  
  const ProfilePage({required this.user});
  
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      body: CustomScrollView(
        slivers: [
          // Collapsing app bar with profile image
          SliverAppBar(
            expandedHeight: 200,
            pinned: true,
            flexibleSpace: FlexibleSpaceBar(
              background: Stack(
                fit: StackFit.expand,
                children: [
                  Image.network(user.coverImageUrl, fit: BoxFit.cover),
                  Container(
                    decoration: BoxDecoration(
                      gradient: LinearGradient(
                        begin: Alignment.topCenter,
                        end: Alignment.bottomCenter,
                        colors: [
                          Colors.transparent,
                          Colors.black.withOpacity(0.7),
                        ],
                      ),
                    ),
                  ),
                ],
              ),
            ),
          ),
          
          // Profile info overlapping the header using a Stack for proper layout bounds
          SliverToBoxAdapter(
            child: Stack(
              clipBehavior: Clip.none,
              children: [
                const SizedBox(height: 80), // Space for the content below
                Positioned(
                  top: -50,
                  left: 16,
                  right: 16,
                  child: Column(
                    crossAxisAlignment: CrossAxisAlignment.start,
                    children: [
                      CircleAvatar(
                        radius: 50,
                        backgroundColor: Theme.of(context).scaffoldBackgroundColor,
                        child: CircleAvatar(
                          radius: 48,
                          backgroundImage: NetworkImage(user.avatarUrl),
                        ),
                      ),
                      const SizedBox(height: 8),
                      Text(
                        user.name,
                        style: Theme.of(context).textTheme.headlineSmall,
                      ),
                      Text(
                        user.bio,
                        style: Theme.of(context).textTheme.bodyMedium,
                      ),
                    ],
                  ),
                ),
              ],
            ),
          ),
          
          // Content sections
          SliverPadding(
            padding: const EdgeInsets.all(16),
            sliver: SliverList(
              delegate: SliverChildListDelegate([
                _buildSection(context, 'Posts', user.posts),
                const SizedBox(height: 16),
                _buildSection(context, 'Photos', user.photos),
              ]),
            ),
          ),
        ],
      ),
    );
  }
  
  Widget _buildSection(BuildContext context, String title, List items) {
    return Column(
      crossAxisAlignment: CrossAxisAlignment.start,
      children: [
        Text(title, style: Theme.of(context).textTheme.titleMedium),
        const SizedBox(height: 8),
        SizedBox(
          height: 150,
          child: ListView.separated(
            scrollDirection: Axis.horizontal,
            itemCount: items.length,
            separatorBuilder: (_, __) => const SizedBox(width: 8),
            itemBuilder: (context, index) => AspectRatio(
              aspectRatio: 1,
              child: Container(
                decoration: BoxDecoration(
                  color: Colors.grey[200],
                  borderRadius: BorderRadius.circular(8),
                ),
              ),
            ),
          ),
        ),
      ],
    );
  }
}
```

### Example 5: Form with Keyboard Handling

```dart
class LoginForm extends StatefulWidget {
  const LoginForm({super.key});
  
  @override
  State<LoginForm> createState() => _LoginFormState();
}

class _LoginFormState extends State<LoginForm> {
  final _formKey = GlobalKey<FormState>();
  
  @override
  Widget build(BuildContext context) {
    return SafeArea(
      child: SingleChildScrollView(
        // Automatically scroll when keyboard appears
        child: Form(
          key: _formKey,
          child: Padding(
            padding: const EdgeInsets.all(24),
            child: Column(
              crossAxisAlignment: CrossAxisAlignment.stretch,
              children: [
                const SizedBox(height: 48),
                Text(
                  'Welcome Back',
                  style: Theme.of(context).textTheme.headlineMedium,
                ),
                const SizedBox(height: 32),
                TextFormField(
                  decoration: const InputDecoration(
                    labelText: 'Email',
                    prefixIcon: Icon(Icons.email),
                    border: OutlineInputBorder(),
                  ),
                  keyboardType: TextInputType.emailAddress,
                  textInputAction: TextInputAction.next,
                ),
                const SizedBox(height: 16),
                TextFormField(
                  decoration: const InputDecoration(
                    labelText: 'Password',
                    prefixIcon: Icon(Icons.lock),
                    border: OutlineInputBorder(),
                  ),
                  obscureText: true,
                  textInputAction: TextInputAction.done,
                  onFieldSubmitted: (_) => _submit(),
                ),
                const SizedBox(height: 24),
                FilledButton(
                  onPressed: _submit,
                  child: const Padding(
                    padding: EdgeInsets.all(12),
                    child: Text('Sign In'),
                  ),
                ),
              ],
            ),
          ),
        ),
      ),
    );
  }
  
  void _submit() {
    if (_formKey.currentState!.validate()) {
      // Handle submission
    }
  }
}
```

## Common Patterns, Anti-Patterns, and Solutions

### Pattern: Preserving Scroll State in Tabs
When using `TabBarView`, use `AutomaticKeepAliveClientMixin` on tab content widgets to prevent them from being disposed when switching tabs.

```dart
class KeepAliveTabContent extends StatefulWidget {
  @override
  State<KeepAliveTabContent> createState() => _KeepAliveTabContentState();
}

class _KeepAliveTabContentState extends State<KeepAliveTabContent>
    with AutomaticKeepAliveClientMixin {
  @override
  bool get wantKeepAlive => true; // Essential for preserving state

  @override
  Widget build(BuildContext context) {
    super.build(context); // Mandatory call
    return ListView.builder(...);
  }
}
```

| Anti-Pattern | Problem | Solution |
|--------------|---------|----------|
| `ListView` in `Column` | Unbounded constraints exception | Wrap in `Expanded` |
| `Text` overflow | Text exceeds available space | Use `overflow: TextOverflow.ellipsis`, `maxLines`, or `Flexible` |
| Fixed pixel sizes | Doesn't scale across devices | Use `FractionallySizedBox`, `AspectRatio`, or percentages |
| Deep nesting | Poor performance, hard to read | Extract into separate widgets |
| Ignoring SafeArea | Content hidden by notches | Wrap in `SafeArea` |
| `SingleChildScrollView` with `ListView` | Nested scroll conflict | Use `ListView` alone or `SingleChildScrollView` with `Column` |
| Missing `const` constructors | Unnecessary rebuilds | Add `const` to stateless widgets |

## Performance Tips

1. **Use `const` constructors** wherever possible to enable widget reuse.
2. **Avoid `IntrinsicWidth`/`IntrinsicHeight`** - they're expensive (O(n²) layout passes).
3. **Use `ListView.builder`** for long lists (lazy loading).
4. **Extract widgets** into separate classes with `const` constructors.
5. **Use `RepaintBoundary`** for complex widgets that repaint frequently.
6. **Avoid deep widget trees** - extract into smaller, reusable components.
