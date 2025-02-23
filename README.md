You're right. Let's add a simple overlap avoidance while keeping the code relatively straightforward. Here's a modified version:

class DetectionOverlay : View {
    private val usedPositions = mutableSetOf<RectF>()
    private val offset = 20f
    
    private fun findLabelPosition(box: RectF, label: String, textBounds: Rect): Point {
        var textX: Float
        var textY = box.centerY()
        
        // Initial X position
        textX = if (box.right + textBounds.width() + offset >= width) {
            box.left - textBounds.width() - offset  // Left side if near screen edge
        } else {
            box.right + offset  // Right side by default
        }
        
        // Create rectangle for this label
        val labelRect = RectF(
            textX,
            textY - textBounds.height(),
            textX + textBounds.width(),
            textY + textBounds.height()
        )
        
        // Check for overlaps and adjust Y position if needed
        while (hasOverlap(labelRect)) {
            // Move label down and update rectangle
            textY += textBounds.height() + offset
            
            // If we're going off screen bottom, start from top
            if (textY + textBounds.height() > height) {
                textY = 0f + textBounds.height()
                // Try other side of box if still overlapping
                if (textX == box.right + offset) {
                    textX = box.left - textBounds.width() - offset
                }
            }
            
            labelRect.set(
                textX,
                textY - textBounds.height(),
                textX + textBounds.width(),
                textY + textBounds.height()
            )
        }
        
        // Add final position to used positions
        usedPositions.add(labelRect)
        
        return Point(textX, textY)
    }
    
    private fun hasOverlap(rect: RectF): Boolean {
        for (used in usedPositions) {
            if (RectF.intersects(used, rect)) {
                return true
            }
        }
        return false
    }

    override fun onDraw(canvas: Canvas) {
        super.onDraw(canvas)
        
        // Clear previous positions
        usedPositions.clear()
        
        boundingBoxes.forEach { (box, label) ->
            // Draw bounding box
            canvas.drawRect(box, boundingBoxPaint)
            
            // Calculate text bounds
            val textBounds = Rect()
            textPaint.getTextBounds(label, 0, label.length, textBounds)
            
            // Get non-overlapping position for label
            val position = findLabelPosition(box, label, textBounds)
            
            // Draw label
            canvas.drawText(label, position.x, position.y, textPaint)
            
            // Optionally draw a line connecting label to box
            canvas.drawLine(
                if (position.x < box.left) box.left else box.right,
                box.centerY(),
                position.x + if (position.x < box.left) textBounds.width() else 0f,
                position.y,
                textPaint
            )
        }
    }
}

This simpler approach still:
1. Tries to place labels on the right or left of boxes
2. If there's overlap, moves the label down
3. If we run out of vertical space, switches to the other side of the box
4. Keeps track of used positions to avoid overlaps
5. Optionally draws connecting lines to boxes

Key improvements:
- Simple vertical stacking to avoid overlaps
- Switches sides when running out of space
- Still handles screen edges
- Much simpler than the complex positioning algorithm
- Added connecting lines to make it clear which label belongs to which box

The main strategy is:
1. Try right side first (unless near screen edge)
2. If labels overlap, stack them vertically
3. If run out of vertical space, try left side
4. Keep track of used positions to prevent any overlaps

Would you like me to:
1. Add margins or padding adjustments?
2. Modify the connecting line style?
3. Add any other simple positioning strategies?
