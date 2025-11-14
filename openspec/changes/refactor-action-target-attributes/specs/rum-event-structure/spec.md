# RUM Event Structure Specification Changes

## MODIFIED Requirements

### Requirement: Action Event Serialization Structure
RUM Action events SHALL serialize target-related attributes within the `action.target` object and gesture-related attributes within the `action.gesture` object, maintaining a logical hierarchical structure.

#### Scenario: Action with target attributes serialization
- **WHEN** an Action event contains target attributes (classname, resource_id, title, etc.)
- **THEN** these attributes SHALL be serialized within the `action.target` object
- **AND** the attributes SHALL NOT appear at the root level of the JSON

#### Scenario: Action with gesture attributes serialization  
- **WHEN** an Action event contains gesture attributes (direction, from_state, to_state)
- **THEN** these attributes SHALL be serialized within the `action.gesture` object
- **AND** the attributes SHALL NOT appear at the root level of the JSON

#### Scenario: Action target object structure
- **WHEN** serializing an Action event with target information
- **THEN** the `action.target` object SHALL contain:
  - `name` (required): target display name
  - `classname` (optional): target view class name
  - `resource_id` (optional): target resource identifier
  - `title` (optional): target title text
  - `selected` (optional): selection state for selectable components
  - `role` (optional): semantic role for accessibility

#### Scenario: Action gesture object structure
- **WHEN** serializing an Action event with gesture information  
- **THEN** the `action.gesture` object SHALL contain:
  - `direction` (optional): gesture direction (up, down, left, right)
  - `from_state` (optional): initial state before gesture
  - `to_state` (optional): final state after gesture

#### Scenario: Action parent target information
- **WHEN** an Action event includes parent container information
- **THEN** the `action.target.parent` object SHALL contain:
  - `index` (optional): index within parent container
  - `classname` (optional): parent container class name  
  - `resource_id` (optional): parent container resource identifier

#### Scenario: Backward compatibility consideration
- **WHEN** upgrading to the new Action event structure
- **THEN** existing attribute collection mechanisms SHALL remain unchanged
- **AND** only the final JSON serialization structure SHALL change
- **AND** RumAttributes constants SHALL maintain the same string values for API compatibility

## REMOVED Requirements

### Requirement: Known Attributes Extraction for Action Properties
**Reason**: The extractKnownAttributes mechanism that moves action.target.* and action.gesture.* attributes to the root level creates inconsistent data structure.

**Migration**: These attributes will now be properly nested within their respective parent objects (action.target, action.gesture) during the initial event construction phase rather than being extracted during serialization.
