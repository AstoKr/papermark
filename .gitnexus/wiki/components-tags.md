# components — tags

# AddTagsModal Component

A modal dialog component for creating and editing tags. Provides a form interface for tag name, color selection, and optional description with built-in validation, plan-based access control, and loading states.

## Purpose

Tags enable users to organize and categorize links. This modal serves as the primary interface for:

- **Creating new tags** with a name, color, and optional description
- **Editing existing tags** by pre-populating form fields with current values
- **Enforcing plan limits** by prompting free users to upgrade when exceeding 5 tags

## Props

```typescript
interface AddTagsModalProps {
  open: boolean;
  setMenuOpen: (open: boolean) => void;
  children?: React.ReactNode;
  tagForm: TagFormState;
  setTagForm: React.Dispatch<React.SetStateAction<TagFormState>>;
  handleSubmit: FormEventHandler<HTMLFormElement> | undefined;
  tagCount?: number;
}
```

| Prop | Type | Description |
|------|------|-------------|
| `open` | `boolean` | Controls modal visibility |
| `setMenuOpen` | `(open: boolean) => void` | Callback to toggle modal state |
| `children` | `React.ReactNode` | Optional trigger element for `DialogTrigger` |
| `tagForm` | `TagFormState` | Current form values including `id` (for edit mode) |
| `setTagForm` | `Dispatch` | State setter for form updates |
| `handleSubmit` | `FormEventHandler` | External form submission handler |
| `tagCount` | `number \| undefined` | Total tags owned by the team |

### TagFormState Shape

```typescript
{
  color: TagColorProps;
  name: string;
  description: string | null;
  loading: boolean;
  id?: string;  // Present when editing an existing tag
}
```

## Form Behavior

### Mode Detection

The component operates in two modes based on `tagForm.id`:

- **Create mode**: `id` is undefined — dialog title shows "Create Tag", button shows "Create Tag"
- **Edit mode**: `id` is present — dialog title shows "Edit Tag", button shows "Save Changes"

### Validation

Form submission is disabled unless `isFormValid` evaluates to true:

```typescript
const isFormValid =
  tagForm.name.length >= 3 &&      // Name requires minimum 3 characters
  !!tagForm.color &&               // Color selection is required
  (!tagForm.id || hasChanged);     // For edits, at least one field must differ from initial values
```

### Dirty State Tracking

The component tracks whether the form has been modified since initial load using a `useRef` to store initial values. This prevents unnecessary save operations when editing tags without making changes.

```typescript
const initialValues = useRef(tagForm);

const hasChanged =
  tagForm.name !== initialValues.current.name ||
  tagForm.color !== initialValues.current.color ||
  tagForm.description !== initialValues.current.description;
```

## Plan Gating

Free plan users are limited to 5 tags. When `tagCount >= 5` and `isFree` is true, the modal renders an `UpgradePlanModal` instead of the standard dialog:

- **Trial users** → Prompted to upgrade to Business plan
- **Other free users** → Prompted to upgrade to Pro plan

```typescript
if (isFree && tagCount >= 5) {
  return (
    <UpgradePlanModal
      clickedPlan={isTrial ? PlanEnum.Business : PlanEnum.Pro}
      trigger={"create_tag"}
    >
      <Button>Upgrade to Create Tags</Button>
    </UpgradePlanModal>
  );
}
```

## Component Structure

```
AddTagsModal
├── Dialog (wrapper)
│   ├── DialogTrigger (children passed in as trigger)
│   └── DialogContent
│       ├── DialogHeader
│       │   ├── DialogTitle (Create Tag / Edit Tag)
│       │   └── DialogDescription
│       ├── Form
│       │   ├── Input (tag name)
│       │   ├── ToggleGroup (color selection)
│       │   │   └── ToggleGroupItem × N (from COLORS_LIST)
│       │   └── Textarea (description)
│       └── DialogFooter
│           └── Button (submit)
└── UpgradePlanModal (conditional, for free plan limit)
```

## Color Selection

Color options are sourced from `COLORS_LIST` (imported from `@/components/links/link-sheet/tags/tag-badge`). Each color renders as a `ToggleGroupItem` with:

- Display of the color name
- Dynamic ring styling when selected: `ring-{color}-500 ring-2`
- Color-specific text styling when active

## Usage Example

```tsx
import { AddTagsModal } from "@/components/tags/add-tag-modal";

function TagSettings() {
  const [open, setOpen] = useState(false);
  const [tagForm, setTagForm] = useState<TagFormState>({
    name: "",
    color: null,
    description: null,
    loading: false,
  });

  const handleSubmit = (e: FormEvent<HTMLFormElement>) => {
    e.preventDefault();
    // Handle tag creation/update
  };

  return (
    <AddTagsModal
      open={open}
      setMenuOpen={setOpen}
      tagForm={tagForm}
      setTagForm={setTagForm}
      handleSubmit={handleSubmit}
      tagCount={existingTags.length}
    >
      <Button onClick={() => setOpen(true)}>+ Add Tag</Button>
    </AddTagsModal>
  );
}
```

## External Dependencies

| Dependency | Purpose |
|-------------|---------|
| `usePlan` | Checks user's subscription tier and trial status |
| `UpgradePlanModal` | Displays upgrade prompts for free plan users |
| `COLORS_LIST` | Predefined color options for tag styling |
| `TagColorProps` | Type definition for valid color values |