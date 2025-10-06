# Modminecraft
import { world, system, BlockLocation, ItemStack } from "@minecraft/server";

world.afterEvents.playerBreakBlock.subscribe(event => {
  const { block, player, dimension } = event;

  const logIds = [
    "minecraft:oak_log",
    "minecraft:birch_log",
    "minecraft:spruce_log",
    "minecraft:jungle_log",
    "minecraft:acacia_log",
    "minecraft:dark_oak_log",
  ];

  if (!logIds.includes(block.typeId)) return;

  const heldItem = player.getComponent("minecraft:equippable").getEquipment("mainhand");
  if (!heldItem || !heldItem.typeId.includes("_axe")) {
    player.sendMessage("⚒ ต้องถือขวาน (Axe) ถึงจะใช้พลังนักตัดไม้ได้!");
    return;
  }

  system.run(() => {
    const totalChopped = chopTree(dimension, block.location, logIds);
    reduceDurability(player, heldItem, totalChopped);
    rewardPlayer(player, dimension, block.location, totalChopped);
    player.sendMessage(`🌲 ตัดไม้สำเร็จ ${totalChopped} บล็อก ได้รับ EXP และของรางวัล!`);
  });
});

function chopTree(dimension, startLoc, logIds) {
  const checked = new Set();
  const stack = [startLoc];
  let count = 0;

  while (stack.length > 0) {
    const loc = stack.pop();
    const key = `${loc.x},${loc.y},${loc.z}`;
    if (checked.has(key)) continue;
    checked.add(key);

    const block = dimension.getBlock(loc);
    if (!block || !logIds.includes(block.typeId)) continue;

    block.setType("minecraft:air");
    dimension.spawnItem({ typeId: block.typeId, amount: 1 }, loc);
    count++;

    const neighbors = [
      new BlockLocation(loc.x + 1, loc.y, loc.z),
      new BlockLocation(loc.x - 1, loc.y, loc.z),
      new BlockLocation(loc.x, loc.y + 1, loc.z),
      new BlockLocation(loc.x, loc.y - 1, loc.z),
      new BlockLocation(loc.x, loc.y, loc.z + 1),
      new BlockLocation(loc.x, loc.y, loc.z - 1),
    ];
    stack.push(...neighbors);
  }
  return count;
}

function reduceDurability(player, item, blocksCut) {
  const durabilityComp = item.getComponent("durability");
  if (!durabilityComp) return;

  const oldDamage = durabilityComp.damage;
  const newDamage = oldDamage + blocksCut;
  durabilityComp.damage = Math.min(newDamage, durabilityComp.maxDurability);

  if (durabilityComp.damage >= durabilityComp.maxDurability) {
    player.sendMessage("❌ ขวานของคุณพังแล้ว!");
    player.getComponent("minecraft:equippable").setEquipment("mainhand", undefined);
  } else {
    player.getComponent("minecraft:equippable").setEquipment("mainhand", item);
  }
}

// 🌟 ฟังก์ชันให้รางวัล EXP และของพิเศษ
function rewardPlayer(player, dimension, loc, totalChopped) {
  // ✅ ให้ EXP
  const exp = totalChopped * 2; // บล็อกละ 2 EXP
  player.addExperience(exp);

  // 🎁 สุ่มของรางวัลพิเศษ
  const dropChance = Math.random();
  let dropItem = null;

  if (dropChance < 0.05) {
    dropItem = "minecraft:golden_apple"; // 5%
  } else if (dropChance < 0.15) {
    dropItem = "minecraft:apple"; // 10%
  } else if (dropChance < 0.35) {
    dropItem = "minecraft:oak_sapling"; // 20%
  }

  if (dropItem) {
    dimension.spawnItem(new ItemStack(dropItem, 1), loc);
    player.sendMessage(`🎁 คุณได้รับ ${dropItem.replace("minecraft:", "")}!`);
  }
}
